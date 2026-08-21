---
title: "每日基础技术总结 · 2026-06-03 · MySQL binlog 与 redolog 的两阶段提交"
date: 2026-06-03 08:00:00
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-06-03 · MySQL binlog 与 redolog 的两阶段提交

## 📚 今日主题

> **MySQL binlog 与 redolog 的两阶段提交**（后端基础）

### 1. 核心概念速览
binlog（归档日志，Server 层）与 redo log（重做日志，InnoDB 存储引擎层）是 MySQL 数据持久性与主从复制的基础。两阶段提交（Two-Phase Commit, 2PC）用于协调这两个日志的写入顺序，保证在崩溃恢复时 binlog 与 redo log 所记录的事务状态一致，从而避免主从数据不一致或数据丢失。其本质是将『写入 redo log 并提交』这一动作拆分为 prepare 与 commit 两个阶段，利用 redo log 的 prepare 状态作为协调点，使 binlog 的写入成为事务是否可提交的最终仲裁。在 MySQL 整体架构中，它是连接存储引擎事务能力与 Server 层复制能力的关键协议；专业工程师必须掌握它，因为任何涉及数据一致性、崩溃恢复、主从同步、备份恢复的问题最终都会追溯到该机制的边界与缺陷。

### 2. 底层原理剖析
事务执行流程如下：
1. 事务执行过程中，InnoDB 先修改内存中的 buffer pool，并生成 redo log 记录，写入 redo log buffer。
2. 事务提交时，进入两阶段提交：
   Phase 1（prepare）：InnoDB 将 redo log 刷入磁盘（通常由 innodb_flush_log_at_trx_commit=1 控制），并标记该事务状态为 prepare。此阶段 redo log 已经持久化，但事务尚未真正提交。
   Phase 2（commit）：Server 层将该事务的 binlog 写入磁盘（由 sync_binlog=1 控制），然后通知 InnoDB 将事务状态标记为 commit（此时仅在 redo log 中写入一个 commit 标记，无需再刷盘）。
崩溃恢复时，扫描最后一个 redo log 中 prepare 与 commit 的记录：
- 若 redo log 中既有 prepare 又有 commit，事务已完整提交，直接重放。
- 若 redo log 中只有 prepare 而无 commit，则检查对应的 binlog 是否完整存在（binlog 中有一个 XID 事件与事务对应）。若 binlog 存在且完整，则事务在 binlog 中已记录，需重新提交（commit）；若 binlog 不存在或不完整，则回滚事务。
此机制确保 binlog 与 redo log 中事务状态永远一致：binlog 有记录则 redo log 必已 commit，binlog 无记录则 redo log 必回滚。
对比前端已有概念：可类比为『数据库存储引擎是操作系统的文件系统，binlog 是应用层的操作日志，redo log 是文件系统内部的 journal』。两阶段提交类似于分布式系统中的原子提交协议（如 Paxos 提交阶段的简化版），但协调者是 MySQL Server 自身。与前端 TypeScript 的 interface 与 Java 的 interface 区别不同——那只是类型系统与多继承语义的差异，而这里的两个日志是不同层级的持久化组件，协调它们的是事务原子性在跨组件间的扩展。

### 3. 基础代码与实战验证
```text
无法用纯 SQL 直接触发两阶段提交，它是 MySQL 内部机制。但可通过以下方式验证其存在与行为：

-- 1. 查看 redo log 状态（需启用 innodb_monitor_enable）
SHOW ENGINE INNODB STATUS;  -- 观察 Log sequence number 与 Log flushed up to

-- 2. 查看 binlog 格式与同步策略
SHOW VARIABLES LIKE 'binlog_format';      -- ROW / STATEMENT / MIXED
SHOW VARIABLES LIKE 'sync_binlog';        -- 每次事务提交后是否刷盘
SHOW VARIABLES LIKE 'innodb_flush_log_at_trx_commit'; -- 0/1/2 控制 redo log 刷盘时机

-- 3. 制造崩溃恢复场景（实验性，生产禁止）
-- 步骤 A：设置 autocommit=0，执行 INSERT，但不提交。
-- 步骤 B：kill -9 mysqld（模拟崩溃），此时 redo log 中只有 prepare（因为未提交不会进入两阶段提交，事务直接回滚）。
-- 步骤 C：启动 MySQL，查看数据——行不存在，说明 redo log 的未提交事务被回滚。
-- 步骤 D：设置 sync_binlog=0 与 innodb_flush_log_at_trx_commit=0（降低刷盘频率），执行事务并提交，随后立即 kill -9。
-- 步骤 E：重启后可能发生：binlog 中无该事务但 redo log 已提交（数据丢失），或 binlog 有但 redo log 未提交（数据不一致）。这正是两阶段提交要避免的情况，因此生产中必须设置两者均为 1。

-- 4. 通过 mysqlbinlog 查看 binlog 中的 XID 事件（XID 就是两阶段提交的协调凭证）
mysqlbinlog --base64-output=DECODE-ROWS -v mysql-bin.000001 | grep -A2 'XID'
```

### 4. 常见误区与进阶思考
误区1：认为两阶段提交是『先写 redo log 再写 binlog』或『先写 binlog 再写 redo log』。实际是：先写 redo log（prepare），再写 binlog，最后写 redo log（commit）。崩溃恢复时以 binlog 是否完整作为最终裁决，而非 redo log 的 prepare 状态。若理解成简单的顺序写，会无法解释崩溃恢复中 binlog 存在但 redo log 缺失 commit 标记时为何要补提交。
误区2：认为 sync_binlog=1 和 innodb_flush_log_at_trx_commit=1 可以随意设置以提升性能。这是安全与性能的权衡点：只有两者都为 1 时，两阶段提交才能保证不丢事务；任何一方为 0，崩溃时都可能出现 binlog 与 redo log 状态不一致，导致主从复制错误或数据丢失。很多工程师为了性能关闭刷盘，却忽略了系统恢复后需要人工介入修复一致性。

思考题：在崩溃恢复时，如果 redo log 中某事务处于 prepare 状态，且对应 binlog 存在但该 binlog 事件尚未被从库消费，此时主库恢复后该事务被提交，但从库连接时能否感知到这个已提交的事务？请结合 binlog 的写入顺序与 dump 线程的读取位置分析，说明两阶段提交如何保证从库不会漏掉该事务。
