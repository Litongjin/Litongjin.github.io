---
title: "每日基础技术总结 · 2026-05-28 · MySQL InnoDB 的 MVCC 与 undo log"
date: 2026-05-28 08:00:00
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-05-28 · MySQL InnoDB 的 MVCC 与 undo log

## 📚 今日主题

> **MySQL InnoDB 的 MVCC 与 undo log**（后端基础）

### 1. 核心概念速览
MVCC（Multi-Version Concurrency Control）是 InnoDB 存储引擎实现事务隔离级别（READ COMMITTED 和 REPEATABLE READ）的核心机制。其本质是：不通过加锁阻塞读，而是为每个数据行维护多个历史版本（基于 undo log 构建的版本链），读操作通过一致性视图（read view）判断当前可见的版本，写操作仍使用行锁，从而在保证隔离性的前提下最大化读写并发。它解决的核心问题是『读写互斥』，同时为事务提供快照隔离。在计算机体系中的位置：位于数据库存储引擎层的并发控制子系统，是事务 ACID 中隔离性（I）与一致性（C）的底层支撑。专业工程师必须掌握它，因为事务隔离级别选择、死锁分析、数据一致性排查、长事务性能问题等，本质上都涉及 MVCC 与 undo log 的交互。

### 2. 底层原理剖析
一、行结构基础
InnoDB 聚簇索引行包含隐藏列：DB_TRX_ID（最近修改该行的事务 ID）、DB_ROLL_PTR（指向 undo log 中前一个版本记录）、DB_ROW_ID（可选）。
二、undo log 与版本链
undo log 记录更新前的镜像。插入操作产生 insert undo（仅回滚用），更新/删除产生 update undo（回滚和 MVCC 都用）。更新时，新版本行通过 DB_ROLL_PTR 指向旧版本，形成从最新到最旧的逻辑版本链。
三、Read View 可见性算法
Read View 包含：活跃事务 ID 集合 active_set，最小活跃事务 ID min_trx_id，下一事务 ID 上限 max_trx_id。判断某行版本 v（其事务 ID 为 trx_id）是否可见：
1. trx_id == 当前事务 ID -> 可见（自己改的）。
2. trx_id < min_trx_id -> 该版本在 Read View 创建前已提交，可见。
3. trx_id >= max_trx_id -> 该版本在 Read View 创建后才开始，不可见。
4. trx_id 在 active_set 中 -> 未提交，不可见。
5. 其他情况（trx_id 介于 min 与 max 之间且不在 active_set）-> 已提交，可见。
若不可见，则通过 v.roll_ptr 沿版本链回溯到更旧版本，重复判断。
四、隔离级别与 Read View 创建时机
READ COMMITTED：每次 SELECT 都新建 Read View，故能看到其他事务新提交的版本。
REPEATABLE READ：事务内第一次 SELECT 创建 Read View，后续快照读复用，故同一事务多次读取结果一致。
五、与前端概念的对比
MVCC 的版本链类似于前端 Git 的 commit 历史：每个 commit 有 parent 指针，分支指针类似 Read View，checkout 到某个 commit 即查看历史快照。也类似于不可变状态管理（如 Redux）中每次更新产生新 state 并通过引用回溯。但本质差异：Git/Redux 是应用层显式版本管理，MVCC 是存储引擎内部自动维护的隐式多版本，且通过 undo log 存储增量逆向操作而非完整快照，并需要 purge 线程回收过期版本。

### 3. 基础代码与实战验证
```text
-- 假设表 t(id INT PRIMARY KEY, val VARCHAR(10))，初始值 (1, 'old')
-- 设置隔离级别为 REPEATABLE READ（默认）

-- 会话 A（事务 T1）
START TRANSACTION;                    -- 分配事务 ID 例如 100
UPDATE t SET val='new' WHERE id=1;    -- 生成新版本，DB_TRX_ID=100，旧版本写入 undo log，DB_ROLL_PTR 指向旧版本
-- 此时不提交

-- 会话 B（事务 T2）
START TRANSACTION;                    -- 分配事务 ID 例如 101
-- 第一次 SELECT 创建 Read View，active_set={100}, min_trx_id=100, max_trx_id=102
SELECT val FROM t WHERE id=1;         -- 行版本 trx_id=100 在 active_set 中不可见，沿 roll_ptr 找到 trx_id=99（已提交）的版本，返回 'old'

-- 会话 A 提交
COMMIT;

-- 会话 B 再次 SELECT（复用第一次的 Read View）
SELECT val FROM t WHERE id=1;         -- 仍返回 'old'，因为 Read View 没有变化，即使 T1 已提交

-- 会话 B 执行 UPDATE（当前读，读取最新已提交版本，忽略 Read View）
UPDATE t SET val='updated' WHERE id=1; -- 当前读读取到 T1 提交的 'new'，然后生成新版本 DB_TRX_ID=101，旧版本再次进入 undo log

-- 会话 B 再 SELECT
SELECT val FROM t WHERE id=1;         -- 行版本 trx_id=101 等于当前事务 ID，可见，返回 'updated'

-- 会话 B 提交
COMMIT;
```

### 4. 常见误区与进阶思考
误区 1：认为 MVCC 意味着所有读操作都不加锁。实际上，MVCC 只针对快照读（普通 SELECT）。对于当前读（SELECT ... FOR UPDATE、UPDATE、DELETE），InnoDB 仍会加行锁，在 REPEATABLE READ 下还可能使用 next-key lock 防止幻读，从而产生阻塞和死锁。
误区 2：认为 undo log 只用于事务回滚。undo log 同时是 MVCC 版本链的载体。如果长事务持续持有 Read View，导致旧版本不能被 purge，undo log 会不断膨胀，引发空间膨胀和性能下降。因此长事务是 InnoDB 大忌。
思考题：REPEATABLE READ 下，事务 T1 先执行一次快照读（SELECT），随后事务 T2 更新同一行并提交，接着 T1 执行 UPDATE（当前读）再执行 SELECT。请问 T1 的 UPDATE 读取的是哪个版本？UPDATE 后 T1 的 SELECT 能看到什么？为什么？
答案：T1 的 UPDATE 读取的是 T2 提交后的最新版本，因为当前读总是读取最新已提交版本，不遵循旧 Read View。UPDATE 后，T1 自己产生新版本（DB_TRX_ID = T1 的 ID）。后续 SELECT 是快照读，但由于版本的事务 ID 等于当前事务 ID，T1 能看到自己的更新。这体现了 MVCC 中『当前读』与『快照读』的路径分离。
