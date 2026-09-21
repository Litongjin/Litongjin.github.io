---
title: "每日基础技术总结 · 2024-08-14 · MySQL redo log 与 binlog 的两阶段提交"
date: 2024-08-14 20:00:00
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-08-14 · MySQL redo log 与 binlog 的两阶段提交

## 📚 今日主题

> **MySQL redo log 与 binlog 的两阶段提交**（后端基础）

### 1. 核心概念速览
核心概念：MySQL InnoDB 存储引擎通过 Redo Log（重做日志）实现 Crash-Safe，确保事务的持久性（Durability）；Binlog（二进制日志）实现 Logical Backup 和主从复制，保障数据的逻辑一致性。两阶段提交（2PC, Two-Phase Commit）是协调这两类日志写入顺序的协议机制。
本质作用：解决在系统崩溃恢复场景下，因 Redo Log 与 Binlog 写入时序不一致导致的‘数据状态矛盾’问题。例如：Redo 已写但 Binlog 未写导致重启后数据回滚丢失业务逻辑；或 Binlog 已写但 Redo 未写导致主从数据不一致。
体系位置：关系型数据库ACID特性的基石，分布式系统中 Saga/TCC 等补偿模式的局部微观原型。专业工程师必须掌握，因为它是理解‘最终一致性’如何落地为‘物理不可变性’的关键路径，也是排查数据错乱、主从延迟的根本依据。

### 2. 底层原理剖析
运行机制详解：
1. 准备阶段 (Prepare)：InnoDB 将事务修改记录写入 redo log buffer，并 flush 到 redo log 文件中，状态标记为 prepare。此时 redo log 仅包含物理页修改指令，尚未关联 binlog。
2. 写入 binlog：Server 层执行 binlog 写操作。若此时失败，redo log 仍为 prepare 状态，InnoDB 重启时可识别出该事务未完成，从而回滚。
3. 提交阶段 (Commit)：binlog 写入成功后，InnoDB 将 redo log 状态改为 commit，并刷盘。

对比前端概念（TypeScript vs JavaScript）：
- Redo Log 类似 TypeScript 编译后的 .js 文件（物理内存布局/磁盘块），直接对应硬件操作，不可变且高效。
- Binlog 类似 TS 源码中的类型定义或 AST 结构（逻辑语义），对外暴露接口供其他服务（如 Slave 节点）消费。
- 两阶段提交类似于编译检查通过后才生成产物。若编译报错（Binlog 失败），则不产出 js 文件（Redo 标记为 abort/prepared but not committed，可安全回滚）；若产物生成成功（Binlog 成功），则确认类型无误（Redo 标记为 committed）。

流程图伪代码：
BEGIN TRAN;
UPDATE table SET x=1;
-- InnoDB: redo_log_buffer -> disk (state: PREPARE)
-- Server: binlog -> disk (success/fail?)
IF binlog_failed THEN
  GOTO ROLLBACK; // 重启时检测到 PREPARE 且无 COMMIT，视为非法事务，回滚
ELSE IF binlog_success THEN
  -- InnoDB: redo_log -> disk (state: COMMIT)
END IF;

### 3. 基础代码与实战验证
```text
// 伪代码展示 InnoDB 内部对 redo log 状态的流转控制逻辑
// 注意：实际 MySQL 源码为 C++，此处为逻辑抽象

struct RedoLogEntry {
    char status; // ENUM: PREPARED, COMMITTED, ABORTED
    uint64_t lsn; // Log Sequence Number
};

void commit_transaction(Transaction* t) {
    // 1. 准备阶段：写入 redo log 并强制落盘（fsync），确保即使崩溃也能恢复
    write_redo_log_to_disk(t->redox_logs);
    set_redo_status(&t->redox_logs.last_entry, PREPARED);
    force_flush_io(); // 关键：os_file_fsync() 

    // 2. 中间阶段：写入 binlog
    // 此步骤由 Server 层处理，若在此处异常抛出，事务仍处于 PREPARED 状态
    try {
        server_layer.write_binlog(t->sql_statements);
    } catch (WriteError& e) {
        // 若 binlog 写入失败，redo log 保持 PREPARED 状态
        // 重启 recovery 时，根据规则：PREPARED + no binlog link = 回滚
        return rollback(t);
    }

    // 3. 提交阶段：更新 redo log 状态为 COMMITTED 并落盘
    set_redo_status(&t->redox_logs.last_entry, COMMITTED);
    force_flush_io(); 
    // 此时事务真正完成，binlog 和 redo log 均保证一致性
}
```

### 4. 常见误区与进阶思考
误区 1：认为 'innodb_commit_on_concurrent_commit' 选项开启后就不需要两阶段提交了。实际上，该优化只是允许多个线程同时调用 commit，底层的 Prepare -> Binlog -> Commit 顺序逻辑不变，除非使用全局锁或特定版本特性，否则仍需保证原子性。

误区 2：混淆 Redo Log 和 Undo Log 的作用。Redo 是用于崩溃恢复（Crash-Safe）的‘正向’重做机制；Undo 是用于事务回滚（Rollback）和 MVCC 多版本控制的‘逆向’撤销机制。两阶段提交只涉及 Redo 和 Binlog 的协调，与 Undo 无关。

深度思考题：
假设 MySQL 主库在写完 binlog 后、修改 redo log status 为 COMMITTED 前发生宕机。Slave 节点已经收到了这条 SQL 并执行了（基于 binlog 的复制机制）。请问：主库重启后，InnoDB 会如何处理这条处于 PREPARED 状态的事务？Slave 的数据是否会发生永久错误？请结合 MTS（Multi-Threaded Slave）或半同步复制机制进一步推导数据一致性的边界。
