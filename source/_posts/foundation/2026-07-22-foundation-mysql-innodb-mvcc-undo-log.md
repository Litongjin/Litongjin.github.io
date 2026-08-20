---
title: "每日基础技术总结 · 2026-07-22 · MySQL InnoDB 的 MVCC 可见性判断与 undo log 版本链"
date: 2026-07-22 08:00:00
categories: [技术分享]
tags: ["技术分享", "架构与设计"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-22 · MySQL InnoDB 的 MVCC 可见性判断与 undo log 版本链

## 📚 今日主题

> **MySQL InnoDB 的 MVCC 可见性判断与 undo log 版本链**（架构与设计）

### 1. 核心概念速览
MVCC（Multi-Version Concurrency Control）是 InnoDB 在 READ COMMITTED 与 REPEATABLE READ 隔离级别下实现非锁定一致性读的机制。其本质是：通过 undo log 构建历史版本链，使事务在读取时无需加锁即可获得某个一致性快照，从而在读写不互斥的前提下保证隔离性。MVCC 解决的核心问题是：在并发事务中，如何让读操作不阻塞写操作、写操作不阻塞读操作，同时满足事务隔离性要求。它位于数据库存储引擎层，属于事务并发控制的核心实现，与锁机制共同构成 InnoDB 的并发控制体系。专业工程师必须掌握它，因为它是理解隔离级别、死锁分析、性能调优、以及分布式事务中快照隔离（SI/SSI）的基础，也是从应用层进入数据库内核层的关键桥梁。

### 2. 底层原理剖析
InnoDB 的 MVCC 依赖三个核心结构：隐藏列、undo log 版本链、ReadView。
1. 隐藏列：每行数据包含 DB_TRX_ID（最近修改该行的事务ID）、DB_ROLL_PTR（指向 undo log 中该行前一版本记录的指针）、DB_ROW_ID（可选，无主键时生成）。
2. undo log 版本链：每次 UPDATE/DELETE 操作，InnoDB 将旧版本数据写入 undo log，并通过 DB_ROLL_PTR 将当前数据行的旧版本链接起来，形成从最新版本到最旧版本的链表。INSERT 的 undo log 在事务提交后通常可清理，因为不再需要。
3. ReadView（一致性视图）：在快照读开始时生成，包含：creator_trx_id（创建该视图的事务ID）、m_ids（活跃事务ID列表）、min_trx_id（活跃事务中最小ID）、max_trx_id（下一个待分配的事务ID，即当前已分配的最大事务ID+1）。
可见性判断规则：对于某个版本的 DB_TRX_ID（记为 trx_id）：
- 若 trx_id == creator_trx_id，则本事务修改的版本可见。
- 若 trx_id < min_trx_id，则该版本在 ReadView 生成前已提交，可见。
- 若 trx_id >= max_trx_id，则该版本在 ReadView 生成后开启的事务所修改，不可见。
- 若 min_trx_id <= trx_id < max_trx_id，则检查 trx_id 是否在 m_ids 中；若在，则该版本由未提交事务修改，不可见；若不在，则该版本在 ReadView 生成前已提交，可见。
判断过程从版本链头部（当前最新版本）开始，若当前版本不可见，则通过 DB_ROLL_PTR 沿 undo log 链回溯，直到找到可见版本或到达链尾。
与前端概念的对比：可类比 JavaScript 原型链与作用域链。原型链用于属性查找，若当前对象无属性则沿 [[Prototype]] 链向上查找，直到找到或到达 null；MVCC 版本链用于行版本查找，若当前版本不可见则沿 DB_ROLL_PTR 链向上查找，直到找到可见版本或到达链尾。更精准的对比是：ReadView 相当于一次快照的“作用域边界”，它捕获了生成时刻的可见事务集合，后续事务的修改不在该作用域内，从而保证一致性。这与 JS 闭包捕获变量环境类似：闭包捕获的是创建时的作用域链，后续外部变量的修改不会影响闭包内的旧值。
不同隔离级别的差异：READ COMMITTED 每次普通 SELECT 都会生成新的 ReadView，因此每次读到的都是当前已提交的最新数据；REPEATABLE READ 只在第一次快照读时生成 ReadView，后续所有快照读复用该视图，从而保证可重复读。

### 3. 基础代码与实战验证
```text
以下以 SQL 会话操作步骤验证 MVCC 可见性，不依赖复杂框架。
-- 准备表结构与数据
CREATE TABLE test (id INT PRIMARY KEY, value INT) ENGINE=InnoDB;
INSERT INTO test VALUES (1, 100);

-- 事务 A（TRX_A）
START TRANSACTION;
-- 此时不生成 ReadView，仅修改数据
UPDATE test SET value = 200 WHERE id = 1;
-- 此时版本链：v2 (value=200, trx_id=TRX_A) -> v1 (value=100, trx_id=较早提交的事务)

-- 事务 B（TRX_B）
START TRANSACTION;
-- 第一次快照读，生成 ReadView
-- ReadView 中 m_ids 包含 TRX_A, TRX_B；min_trx_id=TRX_A；max_trx_id=下一个事务ID
SELECT * FROM test WHERE id = 1;
-- 当前版本 v2 的 trx_id=TRX_A，且 TRX_A 在 m_ids 中，不可见；沿 DB_ROLL_PTR 找到 v1，v1 的 trx_id < min_trx_id，可见；返回 value=100

-- 事务 B 再次查询（REPEATABLE READ 下复用同一个 ReadView）
SELECT * FROM test WHERE id = 1;
-- 即使事务 A 已提交，由于 ReadView 不变，仍返回 value=100

-- 若隔离级别为 READ COMMITTED，事务 B 每次 SELECT 都会重新生成 ReadView；在事务 A 提交后，新 ReadView 中 m_ids 不再包含 TRX_A，当前版本 v2 可见，返回 value=200

-- 验证版本链的伪代码（数据库内部逻辑）
function is_visible(version, read_view):
    trx_id = version.trx_id
    if trx_id == read_view.creator_trx_id:
        return true
    if trx_id < read_view.min_trx_id:
        return true
    if trx_id >= read_view.max_trx_id:
        return false
    if trx_id in read_view.m_ids:
        return false
    else:
        return true

-- 查询时，从版本链头部开始：
cur = latest_version
while cur != NULL:
    if is_visible(cur, read_view):
        return cur.data
    else:
        cur = cur.db_roll_ptr  -- 沿 undo log 回溯
return NULL
```

### 4. 常见误区与进阶思考
1. 误区：认为 MVCC 是“多版本存储”或“历史数据永久保存”。实际上 undo log 仅保存更新前的旧版本用于回滚和可见性判断，一旦没有事务需要这些旧版本（如所有可能引用它的 ReadView 已结束），就会通过 purge 线程清理，并非完整的多版本数据库。
2. 误区：认为 MVCC 完全解决了读写阻塞。MVCC 仅适用于快照读（普通 SELECT），而当前读（SELECT ... FOR UPDATE / LOCK IN SHARE MODE / UPDATE / DELETE）仍使用最新版本并加锁，因此在高并发下仍可能发生锁等待。
思考题：在 REPEATABLE READ 隔离级别下，事务 T1 先 SELECT 某行（生成 ReadView），随后事务 T2 修改并提交该行，接着 T1 再次 SELECT 仍看到旧版本；但若 T1 对该行执行 UPDATE（当前读），会读到最新版本并加锁。请解释为什么当前读不受 ReadView 限制，以及此时 T1 更新后再次 SELECT 该行，能否看到自己更新的值？从版本链与事务ID的分配机制出发分析。
