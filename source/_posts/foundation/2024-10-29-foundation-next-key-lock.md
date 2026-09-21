---
title: "每日基础技术总结 · 2024-10-29 · 行锁、间隙锁与临键锁（Next-Key Lock）"
date: 2024-10-29 20:00:00
categories: [技术分享]
tags: ["技术分享", "数据库与缓存进阶"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-10-29 · 行锁、间隙锁与临键锁（Next-Key Lock）

## 📚 今日主题

> **行锁、间隙锁与临键锁（Next-Key Lock）**（数据库与缓存进阶）

### 1. 核心概念速览
InnoDB存储引擎中MVCC机制下的并发控制核心组件，用于解决读未提交（Read Uncommitted）与读写冲突问题。行锁（Row Lock）：锁定索引记录本身，粒度最小，需通过唯一索引或主键定位，避免全表扫描导致锁升级为表锁。间隙锁（Gap Lock）：锁定索引记录之间的“间隙”，或首尾记录的边界间隙，仅存在于非唯一索引或存在重复值的场景，目的是防止幻读（Phantom Read）。临键锁（Next-Key Lock）：行锁与间隙锁的组合（Left-Open, Right-Closed Interval，即 (key, key]），锁定前一个索引值到当前索引值之间的范围。本质是InnoDB为隔离级别REPEATABLE READ提供的抗幻读解决方案，通过将范围锁定转化为多个离散间隙和记录的集合，确保事务间的数据一致性。掌握它是理解高并发下数据强一致性与性能权衡（如死锁、长事务阻塞）的基础，也是分布式系统中乐观锁/悲观锁在单机数据库层面的映射原型。

### 2. 底层原理剖析
InnoDB的锁算法基于B+树索引结构。默认隔离级别RR下，查询使用Next-Key Lock；更新、删除、插入操作均隐含锁申请。

1. 行锁机制：若WHERE条件命中唯一索引（PRIMARY KEY / UNIQUE），则仅锁定命中的具体行记录（Record Lock），不锁间隙。若命中普通索引，且该索引存在唯一约束，行为同上。若不存在任何索引，或使用覆盖不全的普通索引，由于无法精确定位，InnoDB会隐式提升为表锁（Table Lock）或在无主键时对所有索引加间隙锁，导致性能灾难。

2. 间隙锁机制：当扫描普通索引的非唯一记录时，除了锁定记录本身，还会锁定相邻索引值之间的空间。例如，有序索引值为 10, 20, 30。SELECT * FROM t WHERE id = 25 FOR UPDATE 会锁定区间 (-inf, 10], (10, 20], (20, 30]。注意：(30, +inf) 通常不被包含，除非有大于30的记录被检索。

3. Next-Key Lock合成逻辑：它不是一个独立的锁类型，而是行锁+间隙锁的逻辑集合。公式为：[prev_key, current_key)。例如，对于索引值 20，Next-Key Lock 锁定的是 (prev_index_value, 20]。若无前驱，则为 (-inf, 20]。

对比前端概念：这类似于 TypeScript 中的广义接口（Interface）与狭义类实例的关系。行锁是对‘特定实例’（唯一ID对象）的直接引用锁定；间隙锁是对‘类型定义域内的空白区域’（非唯一ID的可选范围）进行占位锁定，防止其他事务在该区域内‘实例化’新数据，从而破坏当前事务预期的集合完整性（幻读）。

### 3. 基础代码与实战验证
```text
// 假设表 user_id 为主键（唯一），status 为普通索引（非唯一，存在重复值）
// 当前数据: (1, 'A'), (2, 'B'), (5, 'C'), (8, 'D')

-- 场景1：利用主键精确查找 -> 仅产生行锁
BEGIN;
UPDATE users SET status='updated' WHERE id = 5; 
-- 底层：直接命中 B+Tree 叶节点记录 {id:5}，仅锁定该行记录。其他事务可修改 id=6,7 等，不阻塞。
COMMIT;

-- 场景2：利用普通索引非唯一列查找 -> 产生 Next-Key Lock (间隙+行)
BEGIN;
UPDATE users SET status='updated' WHERE status = 'C'; -- 找到 id=5 的记录
-- 底层：命中普通索引 status='C'。由于 status 非唯一，InnoDB 锁定范围：(Previous_Status, 'C']。
-- 若上一记录 status='B'，则锁定区间 ('B', 'C']。即锁定 status 在 (B, C] 范围内的所有潜在记录。
-- 此时，其他事务尝试 INSERT (new_id, 'CB') 或 UPDATE set status='CB' 将被阻塞，因为 'CB' 落在间隙内。
COMMIT;

-- 场景3：无索引查找 -> 退化为表锁或全索引间隙锁
BEGIN;
SELECT * FROM users WHERE first_name = 'John' LOCK IN SHARE MODE;
-- 若无索引，InnoDB 需扫描全表或所有二级索引，对每条扫描过的记录都加上 Next-Key Lock。
-- 后果：极高并发下几乎锁定整张表，严重降低吞吐。
COMMIT;
```

### 4. 常见误区与进阶思考
["误区一：认为 'WHERE 条件用了索引就一定是行锁'。纠正：只有当索引是 PRIMARY KEY 或 UNIQUE 索引，且 WHERE 子句完全匹配该唯一索引时，才是纯行锁。若使用普通索引（即使能定位到单行），只要该列存在重复值的可能性，InnoDB 默认仍会施加 Next-Key Lock 以防止幻读。", '误区二：忽视间隙锁导致的‘假性死锁’。两个事务分别锁定不同的间隙，但都试图插入另一个事务已锁定的间隙内的值，或者按不同顺序请求相邻间隙，极易形成循环等待。在高并发写入场景中，过度依赖普通索引做范围查询是引发生产环境死锁的主要原因。']
