---
title: "每日基础技术总结 · 2026-11-12 · 行锁、间隙锁与临键锁"
date: 2026-11-12 07:02:24
categories: [技术分享]
tags: ["技术分享", "数据库与缓存进阶"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-12 · 行锁、间隙锁与临键锁

## 📚 今日主题

> **行锁、间隙锁与临键锁**（数据库与缓存进阶）

### 1. 核心概念速览
核心概念速览

行锁（Record Lock）是 InnoDB 对单个索引记录对象施加的互斥锁；间隙锁（Gap Lock）是对索引记录之间的开区间或首/尾无穷区间施加的插入禁止锁；临键锁（Next-Key Lock）是记录锁与前一个间隙锁的组合，锁定左开右闭区间。它们解决的问题不是简单的“读同一行”，而是并发场景下当前读与写入导致的一致性与幻读：行锁保护已存在记录的更新权；间隙锁保护索引集合在事务期间的区间不变式，阻止其他事务向该区间插入新记录；临键锁同时保护记录本身和边界区间，是 REPEATABLE READ 下防止范围查询出现幻影行的核心机制。它位于并发控制与隔离级别实现之间：上承 SQL 语义和事务隔离，下接锁表、事务等待队列、MVCC、redo/undo 日志。专业工程师必须掌握，因为后端接口正确性常由锁决策决定：锁等待直接造成 P95/P99 尾延迟、事务超时、5xx 和死锁；不理解锁边界就无法判断 UPDATE/SELECT FOR UPDATE 的真实并发副作用。与前端心智模型相比，React/Redux 的 reducer 原子性来自单线程事件循环，而 InnoDB 是共享状态、可抢占 I/O、多会话并发，必须显式声明锁对象和冲突关系，类似于从“JS 事件循环串行化”切换到“跨进程资源调度器”。

### 2. 底层原理剖析
底层原理剖析

InnoDB 加锁对象首先是索引记录，而不是表或行物理地址；没有可用索引时可能退化为全表扫描锁所有行和间隙。核心流程：
1. 解析事务隔离级别与语句类型；普通一致性读由 MVCC snapshot 返回历史版本，不加锁。
2. 当前读语句（SELECT ... FOR UPDATE / FOR SHARE / UPDATE / DELETE / INSERT 唯一键检查）必须读取最新版本并决定锁集合。
3. 执行计划选定索引；沿 B+Tree 定位记录或区间。
4. 根据索引唯一性、命中/未命中、查询条件范围构造锁：唯一索引等值命中退化为行锁；唯一索引等值未命中退化为间隙锁；非唯一索引等值或范围通常加临键锁覆盖扫描范围，并锁定最后一个不匹配记录的边界；RR 下范围更新可锁到 supremum。
5. 将锁请求插入锁表：每条锁包含 transaction_id、index、lock_range、mode(GAP/RECORD/NEXT_KEY/INSERT_INTENTION)、type(S/X/IS/IX)。
6. 与已有锁做兼容检查；兼容则立即持有，冲突则进入等待队列，按事务顺序或调度规则等待；commit/rollback 释放全部锁。

伪代码：
lock_current_read(statement, index, predicate):
  records = btree_scan(index, predicate)
  if is_unique_index(index) and eq_predicate(predicate):
    for rec in records where exact_match:
      add_lock(rec, RECORD, X or S)
    if no exact_match:
      add_lock(gap_between(pred_value), GAP)
  else:
    for rec in records:
      add_lock((prev_key_gap, rec], NEXT_KEY, X or S)
    add_lock((last_scanned_gap], NEXT_KEY or GAP)
  wait_or_acquire(transaction)

锁兼容关键规则：GAP 锁之间通常互相兼容；X RECORD 与任何 S/X RECORD 冲突；NEXT_KEY 包含 GAP，因此会阻塞 INSERT_INTENTION，也会与其他 NEXT_KEY/RECORD 冲突；SELECT FOR UPDATE 会获取 X next-key 或 X record；SELECT FOR SHARE 获取 S 版本。RR 加 gap/next-key 的目的不是让普通读变成阻塞读，而是让当前读和写看到稳定索引边界，从而阻止幻影插入。RC 不加 gap/next-key（多数普通语句），因此同一范围查询两次可能看到新增记录；但仍可能在唯一键检查、外键检查、扫描更新时加特定 gap。与前端概念对比：TS 接口和 Java 接口都表达契约，但 TS 接口只在类型层存在，运行时可被绕过；InnoDB 锁不是类型契约，而是运行时调度器的强制约束。类似地，React 状态一致性由 reducer 纯函数和事件循环保证；InnoDB 一致性由锁表、undo log 和事务生命周期保证。行锁像单个 key 的 mutex，间隙锁像区间不变式 guard，临键锁像 key 与 interval guard 的组合。

### 3. 基础代码与实战验证
```text
基础代码与实战验证

准备表：
CREATE TABLE t(id INT PRIMARY KEY, c INT, d INT, INDEX idx_c(c));
-- 主键聚簇索引存在，普通二级索引 idx_c 指向主键；锁粒度沿二级索引记录与间隙计算。

场景一：非唯一索引范围 SELECT FOR UPDATE 触发临键锁与间隙阻塞。
-- Session A
BEGIN;
SELECT * FROM t FORCE INDEX(idx_c) WHERE c BETWEEN 10 AND 20 FOR UPDATE;
-- 在 idx_c 上对匹配记录及前序间隙加 X next-key；边界会延伸到最后一个不匹配记录，阻止并发 INSERT c=15/c=25 等落入锁定间隙。

-- Session B
BEGIN;
INSERT INTO t VALUES(101,15,1);
-- INSERT 需要 idx_c=(15) 的插入意向锁；该意向锁与 Session A 的 gap/next-key 冲突，B 被阻塞。
-- 若 Session A COMMIT 或 ROLLBACK，锁释放后 Session B 继续执行。

场景二：唯一索引等值命中退化为行锁。
-- Session A
BEGIN;
SELECT * FROM t WHERE id=1 FOR UPDATE;
-- id 是主键且唯一，命中单条记录时只加 X record lock，不锁相邻间隙；并发事务可以插入 id=2，也可以更新其他记录，但不能更新/删除 id=1。

-- Session B
UPDATE t SET d=99 WHERE id=1;
-- 与 Session A 的 X record lock 冲突；必须等待 A 提交/回滚。

场景三：唯一索引等值未命中退化为间隙锁。
-- Session A
BEGIN;
SELECT * FROM t WHERE id=5 FOR UPDATE;
-- id 唯一但无记录，InnoDB 在相邻索引区间之间加 gap lock；不保护不存在的记录，而是阻止后续向该间隙插入 id=5。

-- Session B
INSERT INTO t VALUES(5,1,1);
-- 插入意向锁与 gap lock 冲突；B 被阻塞；A 回滚后 B 可插入。

场景四：查看锁状态。
SELECT ENGINE_TRANSACTION_ID,LOCK_TYPE,LOCK_MODE,LOCK_STATUS,INDEX_NAME,LOCK_DATA
FROM performance_schema.data_locks;
-- LOCK_MODE 显示 X/GAP 或 X,REC_NOT_GAP；data_locks 反映事务当前持有的锁区间与等待状态，是诊断锁等待和死锁的底层证据。

注意：实际锁范围受索引记录、查询优化、RR/RC、唯一性、扫描方向影响；用 FORCE INDEX 可固定路径，避免优化器改变计划导致锁形态不同。
```

### 4. 常见误区与进阶思考
常见误区与进阶思考

误区一：把 NEXT_KEY LOCK 理解为所有查询都会锁很多行。事实是等值命中唯一索引通常退化为 RECORD LOCK；只有非唯一索引扫描或范围扫描才保留 NEXT_KEY/GAP；未命中的唯一等值才是 GAP LOCK。锁范围由执行计划、索引、隔离级别共同决定，不由 SQL 文本单独决定。

误区二：把 GAP LOCK 当作读写互斥锁。GAP LOCK 主要阻止其他事务的 INSERT 或移动记录进入被锁间隙，GAP LOCK 之间通常兼容；它不会把所有读请求都阻塞。普通 SELECT 在 MVCC 下不加锁，只有当前读才参与锁调度。因此看到事务因 gap 锁等待，通常是插入冲突或当前读更新冲突，而不是简单并发读。

进阶思考题：在 REPEATABLE READ 下，执行 UPDATE t SET d=d+1 WHERE c=10，c 是非唯一索引。锁日志显示持有 idx_c 上 (9,10]、(10,12] 的 X next-key。请推导：为什么 12 即使不等于 10 也被锁住？另一个事务 INSERT (id,c=11) 和 INSERT (id,c=9) 谁会阻塞、谁可能通过？这要求从索引扫描边界、B+Tree 记录顺序、next-key 左开右闭语义和插入意向锁冲突关系回答，而不是只记住范围更新锁两边。
