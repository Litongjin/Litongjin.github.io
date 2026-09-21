---
title: "每日基础技术总结 · 2024-06-15 · MVCC 多版本并发控制与 undo log"
date: 2024-06-15 20:00:00
categories: [技术分享]
tags: ["技术分享", "数据库与缓存进阶"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-06-15 · MVCC 多版本并发控制与 undo log

## 📚 今日主题

> **MVCC 多版本并发控制与 undo log**（数据库与缓存进阶）

### 1. 核心概念速览
MVCC（Multi-Version Concurrency Control）是一种在不引入锁竞争的前提下实现事务隔离性的并发控制机制，其本质是通过空间换时间，利用版本链解决读写冲突。undo log 是 MVCC 的基石，记录数据修改前的旧值版本，用于构建历史版本快照及事务回滚。在计算机体系中，它位于存储引擎层，直接对接磁盘 I/O 与内存 Buffer Pool；对前端工程师而言，理解 MVCC 有助于从“同步阻塞式 API”思维转向“异步非阻塞、最终一致性”的后端架构思维，是掌握高性能数据库内核与分布式系统一致性协议的必备基础。

### 2. 底层原理剖析
1. 核心组件：InnoDB 每行记录包含 Hidden Columns（db_trx_id: 最后修改事务 ID, db_roll_ptr: 指向 undo log 中前一个版本的指针）以及 Read View（读视图，包含活跃事务列表 min_id/max_id 及 array）。
2. 写操作流程：执行 INSERT/UPDATE/DELETE 时，将当前事务 ID 写入行的 db_trx_id，并将旧数据版本追加到 undo log 链表头部，更新 roll_ptr。
3. 读操作流程（RC/RR 隔离级别差异）：
   - SELECT 发起时生成 Read View。
   - 通过 DB_TRX_ID 与 Read View 中的活跃事务 ID 进行可见性算法比对：若行版本的事务 ID < 提交的最小事务 ID，则可见；若在最大事务 ID 之后或存在于活跃列表中，则不可见，需沿 undo log 链表回溯查找上一个版本，重复上述判定，直到找到可见版本或无更多版本。
4. 前端对比：如同 TypeScript 编译时的类型擦除（Emit）生成运行时元数据以支持反射，undo log 即为数据库行记录的‘运行时元数据’，不改变主数据结构布局但提供版本追溯能力。这与前端 React 的 Fiber 架构类似，通过链表结构管理状态历史，而非单一突变状态。

### 3. 基础代码与实战验证
```text
// 伪代码模拟 InnoDB 行记录结构与可见性判断逻辑
struct RowData {
    col_a: int;
    col_b: string;
    hidden_cols: {
        db_trx_id: TransactionID; // 最近修改该行的事务ID
        db_roll_ptr: RollPointer;  // Undo Log 版本链指针
    };
}

struct ReadView {
    min_trx_id: TransactionID; // 创建视图时的最小活跃事务ID
    max_trx_id: TransactionID; // 创建视图时的下一个将要分配的事务ID
    creator_trx_id: TransactionID;
    active_txns: Set<TransactionID>; // 当前活跃事务ID集合
}

function isVersionVisible(row: RowData, view: ReadView): boolean {
    const txId = row.hidden_cols.db_trx_id;
    
    // 1. 如果修改该行的事务ID尚未在视图中可见（未来事务），则不可见
    if (txId >= view.max_trx_id) return false;
    
    // 2. 如果修改该行的事务ID已在视图中完全提交（早于最小活跃ID），则可见
    if (txId < view.min_trx_id) return true;
    
    // 3. 如果处于 [min_trx_id, max_trx_id) 区间，需检查是否仍活跃
    if (view.active_txns.has(txId)) return false;
    
    // 否则视为已提交且可见
    return true;
}

// 查询逻辑：当 isVersionVisible 返回 false 时，
// 令 currentRow = getPreviousVersion(currentRow.hidden_cols.db_roll_ptr)
// 递归调用 isVersionVisible，直至返回 true 或 roll_ptr == NULL
```

### 4. 常见误区与进阶思考
误区：认为 MVCC 完全消除了锁。真相：MVCC 仅优化了普通 SELECT 的锁行为，但在高并发写场景下仍存在间隙锁（Gap Lock）和 Next-Key Lock 以防止幻读，且在 UPDATE/DELETE 时仍需加排他锁（X Lock）保证数据一致性。
进阶思考：在 Repeatable Read (RR) 隔离级别下，为什么第一次 SELECT 生成的 Read View 会被后续 SELECT 复用？而在 Read Committed (RC) 下每次 SELECT 都生成新 Read View？请从事务一致性的粒度与性能开销的角度，推导两者在半自动提交（Auto-commit）与非自动提交事务中的具体行为差异及其对‘幻读’现象的控制边界。
