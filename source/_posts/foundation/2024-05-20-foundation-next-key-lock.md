---
title: "每日基础技术总结 · 2024-05-20 · 行锁、间隙锁与临键锁（Next-Key Lock）"
date: 2024-05-20 20:00:00
categories: [技术分享]
tags: ["技术分享", "数据库与缓存进阶"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-05-20 · 行锁、间隙锁与临键锁（Next-Key Lock）

## 📚 今日主题

> **行锁、间隙锁与临键锁（Next-Key Lock）**（数据库与缓存进阶）

### 1. 核心概念速览
行锁（Record Lock）、间隙锁（Gap Lock）与临键锁（Next-Key Lock）是 InnoDB 存储引擎在可重复读（Repeatable Read）隔离级别下，为实施 MVCC 和防止幻读而设计的锁定机制。本质：行锁锁定索引记录，间隙锁锁定索引记录之间的空隙或范围边界外的开区间，二者结合构成的 Next-Key Lock 锁定的是左开右闭区间 (key, +∞)。解决的核心问题是在并发环境下维持数据的逻辑一致性，特别是防止事务 A 读取数据后，事务 B 插入新数据导致事务 A 再次查询出现‘幻觉’（Phantom Read）。在数据库内核中，这些锁作用于二级索引和聚簇索引的记录级元数据，而非数据页。专业工程师必须掌握以理解高并发场景下的死锁成因、锁粒度优化及性能瓶颈根源，这是从应用层逻辑深入到存储引擎实现的必经之路。

principals": "运行机制基于 B+ 树的索引遍历过程。1. 行锁：当 SQL 通过唯一索引精确匹配一行时，仅对该记录加锁（等价于 Record Lock）。2. 间隙锁：当 SQL 使用范围查询或非唯一索引搜索，或主键不存在时，锁定索引间隙，防止其他事务在间隙内插入数据（Gap Lock）。3. Next-Key Lock = Gap Lock(左) + Record Lock(右)。例如 select * from table where id > 5 for update，若 5 存在，锁定 (5, next_key]；若 5 不存在，锁定 (-inf, 5)。前端对比：Java 接口定义契约，TS 类型系统在编译期进行静态检查；而行锁是运行时资源互斥手段，间隙锁类似 JS 中的 Symbol 创建唯一作用域，Next-Key Lock 则是将对象属性访问（行锁）与作用域保护（间隙锁）结合，确保迭代过程中的状态稳定性。核心差异：前端锁多关注同步原语（Mutex/Semaphore），DB 锁关注数据版本的可见性与隔离性，锁的持有时间贯穿整个事务直至 commit/rollback。"

### 3. 基础代码与实战验证
```text
/* 假设表 t(id int primary key, val int)，当前数据: (10, 1), (20, 2) */

/* 场景 1: 精确命中，行锁 */
/* 事务 A */
START TRANSACTION;
SELECT * FROM t WHERE id = 10 FOR UPDATE; /* 仅锁定 id=10 这条记录 */
/* 事务 B */
INSERT INTO t VALUES (5, 3); -- 成功，因为 5 不在 10 的行锁范围内
DELETE FROM t WHERE id = 20; -- 成功，不同行

/* 场景 2: 范围查询，Next-Key Lock */
/* 事务 A */
START TRANSACTION;
SELECT * FROM t WHERE id > 10 FOR UPDATE; 
/* 底层行为: 对 id > 10 的记录加 Next-Key Lock。对于 id=20，锁定区间 (10, 20]。*/
/* 注意: 如果 id=10 不存在，则锁定 (-inf, 10] */

/* 事务 B 试图插入干扰项 */
INSERT INTO t VALUES (15, 4); /* 阻塞! 15 落在区间 (10, 20] 内，被间隙部分阻挡 */
INSERT INTO t VALUES (30, 5); /* 等待... 取决于实现细节，但通常 (10, 20] 不挡 30，除非有更大的索引 */
/* 注: 上述 INSERT 30 是否阻塞取决于是否存在 30 之后的索引键。若只有 20，(10, +inf) 通常会锁到下一个键或无穷远。在实际验证中，建议测试 INSERT 落在间隙内的行为。 */
```
