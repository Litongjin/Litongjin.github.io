---
title: "每日基础技术总结 · 2026-08-04 · MySQL InnoDB 的 Next-Key Lock：间隙锁与记录锁的加锁区间"
date: 2026-08-04 08:00:00
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-08-04 · MySQL InnoDB 的 Next-Key Lock：间隙锁与记录锁的加锁区间

## 📚 今日主题

> **MySQL InnoDB 的 Next-Key Lock：间隙锁与记录锁的加锁区间**（后端基础）

### 1. 核心概念速览
Next-Key Lock 是 InnoDB 在 REPEATABLE READ 隔离级别下默认采用的索引锁机制，本质是“记录锁 + 间隙锁”的复合体。其加锁区间是一个左开右闭的索引区间 (previous_key, current_key]，既包含当前索引记录本身，也包含当前记录与前一条记录之间的间隙。它通过锁定间隙来阻止其他事务在已扫描范围内插入新记录，从而解决幻读问题。在计算机体系结构中，它属于数据库事务的隔离性实现层，位于存储引擎（InnoDB）内部，是 ACID 中隔离性（I）的关键保障。专业工程师必须掌握它，因为锁区间的大小直接决定事务的并发度、死锁风险和性能瓶颈；对它的理解深度决定了能否正确设计高并发写入方案、排查死锁日志并优化事务隔离级别。

### 2. 底层原理剖析
加锁规则（基于索引扫描）：
1. 每条被扫描到的索引记录，默认被加 Next-Key Lock，锁定区间为 (前一条索引记录值, 当前记录值]。
2. 索引记录之间的“间隙”由后续记录的 Next-Key Lock 中的 Gap Lock 部分覆盖。例如索引值分布为 1、3、5，则潜在 Next-Key Lock 区间为 (-∞,1], (1,3], (3,5], (5,+∞]，其中 +∞ 是 InnoDB 中的伪记录 supremum。
3. 退化规则：
   - 唯一索引等值查询且记录存在：退化为 Record Lock，只锁该记录本身，因为唯一性约束已保证不可能插入相同键值。
   - 唯一索引等值查询且记录不存在：退化为 Gap Lock，锁定一个开区间。
   - 普通索引等值查询：锁定所有匹配记录的 Next-Key Lock，并额外在第一个不匹配记录处加 Gap Lock，以封闭右边界。
   - 范围查询：对扫描触及的每个索引记录加 Next-Key Lock，包括范围结束后的第一个记录（其 Gap Lock 防止边界插入）。

伪代码描述：
for rec in index_scan(condition):
    if rec satisfies condition:
        add_next_key_lock(rec)  # 覆盖 (prev_key, rec_key]
    else:
        add_gap_lock(rec)       # 覆盖 (last_key, rec_key)
        break

对比前端并发控制：前端典型为单线程事件循环，不存在幻读问题；但在 Web Workers 中，SharedArrayBuffer 与 Atomics 提供了多线程同步原语，其锁粒度是单个内存位置，类似于 Record Lock。而 Next-Key Lock 扩展到了“有序索引上的区间”，这是数据库处理范围查询时特有的问题。这类似于 Java 接口与 TS 接口的差异：两者都定义契约，但 Java 接口在编译期强类型约束，TS 接口是结构化子类型约束——同样的“锁”概念，在不同领域有不同的粒度和作用范围。

### 3. 基础代码与实战验证
```text
-- 验证场景：REPEATABLE READ 隔离级别，InnoDB 引擎
CREATE TABLE t (id INT PRIMARY KEY, value VARCHAR(10)) ENGINE=InnoDB;
INSERT INTO t VALUES (1, 'a'), (3, 'b'), (5, 'c');

-- 事务 A：范围查询，触发 Next-Key Lock
START TRANSACTION;
-- 该查询会扫描 id=3、id=5，并继续扫描到 supremum
-- 加锁区间为 (1,3], (3,5], (5, +∞)
SELECT * FROM t WHERE id > 2 FOR UPDATE;

-- 事务 B（在另一个会话执行）：尝试插入 id=2 或 id=4
START TRANSACTION;
-- 插入 id=2：目标落在间隙 (1,3)，被事务 A 的 Gap Lock 阻塞
INSERT INTO t VALUES (2, 'd');
-- 插入 id=4：目标落在间隙 (3,5)，同样被阻塞
INSERT INTO t VALUES (4, 'e');

-- 对比：唯一索引等值查询退化为 Record Lock
-- 事务 C：
START TRANSACTION;
SELECT * FROM t WHERE id = 3 FOR UPDATE;

-- 事务 D（另一会话）：
START TRANSACTION;
-- 插入 id=2 或 id=4 成功，因为 id=3 的记录锁不阻止相邻间隙插入
INSERT INTO t VALUES (2, 'd');
INSERT INTO t VALUES (4, 'e');
-- 但更新 id=3 会被阻塞
UPDATE t SET value='x' WHERE id=3;

注意：以上阻塞行为需要保持事务 A 未提交，并在可重复读隔离级别下测试。
```

### 4. 常见误区与进阶思考
- 误区 1：认为“唯一索引不会产生间隙锁”。实际上，唯一索引等值查询仅在记录存在时退化为记录锁；若记录不存在（例如 WHERE id=2 且表中无 2），则会产生间隙锁，阻止其他事务在 (1,3) 插入 2。范围查询也不会因为索引唯一而避免间隙锁。
- 误区 2：认为“FOR UPDATE 只锁定返回的记录行”。在 RR 隔离级别下，范围查询的 FOR UPDATE 会锁定扫描路径涉及的所有间隙，甚至可能锁定到范围边界之外。如果查询未使用索引，则会退化为全表扫描，导致锁住整个表的所有间隙和记录，严重降低并发度。
- 进阶思考题：在 REPEATABLE READ 下，若事务 A 执行 SELECT * FROM t WHERE id = 3 FOR UPDATE（id 是唯一索引且记录存在），事务 B 可以成功插入 id=2。但若事务 A 改为 SELECT * FROM t WHERE id >= 3 FOR UPDATE，事务 B 插入 id=2 是否会被阻塞？请从加锁区间和幻读定义的角度解释原因。
