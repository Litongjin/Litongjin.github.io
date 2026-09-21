---
title: "每日基础技术总结 · 2026-04-17 · 事务隔离级别：脏读/不可重复读/幻读"
date: 2026-04-17 20:00:00
categories: [技术分享]
tags: ["技术分享", "数据库与缓存进阶"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-04-17 · 事务隔离级别：脏读/不可重复读/幻读

## 📚 今日主题

> **事务隔离级别：脏读/不可重复读/幻读**（数据库与缓存进阶）

### 1. 核心概念速览
事务隔离级别（Isolation Levels）是并发控制的核心机制，用于定义事务在执行过程中对其它事务的可见性约束。其本质是通过锁定或版本一致性检查（MVCC）来解决多用户环境下的数据竞争问题。

1. 脏读（Dirty Read）：读取了未提交的数据。解决：禁止读取未committed版本。
2. 不可重复读（Non-Repeatable Read）：同一事务内多次读取同一行数据，结果不一致（通常由其他事务修改并提交导致）。解决：读取期间加排他锁（写锁）或锁定快照。
3. 幻读（Phantom Read）：同一事务内两次查询范围条件返回的结果集行数不一致（通常由其他事务插入或删除符合条件的新记录导致）。解决：间隙锁（Gap Lock）、Next-Key Lock 或严格快照隔离。

在计算机体系中，它是分布式系统与本地数据库性能/一致性的权衡支点。专业工程师必须掌握，因为它是理解 CAP 定理中 C（一致性）、AP 系统中最终一致性实现差异、以及 AI 训练数据流水线中并发写入安全的基础。

### 2. 底层原理剖析
底层机制主要依赖两种技术栈：锁机制（Lock-based）与多版本并发控制（MVCC, Multi-Version Concurrency Control）。

1. 读已提交（RC, Read Committed）：基于 MVCC，每次 SELECT 获取当前最新的 committed 版本快照。RC 下可避免脏读，但无法避免不可重复读（因为每次查询可能生成新快照）和幻读。
2. 可重复读（RR, Repeatable Read）：默认 InnoDB 隔离级。启动事务时获取一个全局一致的快照（Snapshot），整个事务生命周期内 SELECT 均读取该时间点的快照数据，避免不可重复读。InnoDB 通过 Next-Key Lock 进一步限制插入冲突，从而在绝大多数场景下规避幻读，但未完全从原理上杜绝（如唯一索引非阻塞情况下仍可能受间隙锁影响逻辑边界）。
3. 串行化（Serializable）：强制串行执行，读写互斥，最高隔离性，最低并发度。

与前端概念对比：
- RC vs RR 类似于 JavaScript 中的宏任务（Macrotask）微任务（Microtask）调度差异，或者说 React setState 的同步更新（RR，批次处理）与异步批量更新（RC，最新值访问）的区别。RR 保证的是“视角的一致性”，而非数据的绝对锁定。

### 3. 基础代码与实战验证
```text
-- 伪代码演示 RR 级别下的不可重复读与幻读规避机制
-- 初始化：Table(id PK, val INT)

Session A (Tx1)               | Session B (Tx2)
-----------------------------|-----------------------------------
BEGIN;                         | BEGIN;
SELECT * FROM t WHERE id=1;   |
// 返回 val=10                  |
                               | UPDATE t SET val=20 WHERE id=1;
                               | COMMIT;  -- 会话B提交通常立即生效
SELECT * FROM t WHERE id=1;   |-- Session A 再次查询
// 仍返回 val=10 (MVCC 快照一致)| // RR 下读取旧版本，无不可重复读

-- 幻读模拟
SELECT * FROM t WHERE val > 10;|-- Session A 第一次范围查询，返回空(假设无>10)
                               |
                               | INSERT INTO t VALUES (2, 30);
                               | COMMIT;
SELECT * FROM t WHERE val > 10;|-- Session A 第二次范围查询
// InnoDB Next-Key Lock 会阻止| // 取决于是否触发间隙锁释放/重查
// 典型的幻读在纯 RR+GapLock   | // 逻辑上会被视为“未产生”或
// 下被有效抑制，但在某些非     | // “重新扫描发现”，需视具体引擎
// 阻塞算法下仍可能发生。       | // 实现而定。
```

### 4. 常见误区与进阶思考
1. 误区：认为「可重复读」能完全消除所有幻读。正解：MVCC 解决了大部分不可重复读和一般幻读，但针对插入操作的幻读，InnoDB 是通过 Next-Key Lock 实现的副作用保护，而非纯粹 MVCC 的特性。在其他数据库（如 PostgreSQL）中，RR 级别下依然允许幻读，仅通过 Snapshot 保证列值的稳定。

2. 误区：混淆「锁隔离」与「快照隔离」。正解：SQL Server 早期版本及 Oracle 的 Read Committed 使用共享锁防止脏读和不可重复读，效率较低；而现代主流关系型数据库利用 Undo Log 实现 MVCC，读不加锁，写加锁，二者设计哲学截然不同。

思考题：在高并发热点行更新场景中，为什么将隔离级别从 REPEATABLE READ 降低为 READ COMMITTED 有时能显著提升系统吞吐量？请结合 InnoDB 的 Next-Key Lock 与间隙锁开销进行分析。
