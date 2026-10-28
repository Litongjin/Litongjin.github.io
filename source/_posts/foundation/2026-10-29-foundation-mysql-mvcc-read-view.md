---
title: "每日基础技术总结 · 2026-10-29 · MySQL MVCC实现原理：Read View生成时机与版本链遍历"
date: 2026-10-29 07:06:06
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-29 · MySQL MVCC实现原理：Read View生成时机与版本链遍历

## 📚 今日主题

> **MySQL MVCC实现原理：Read View生成时机与版本链遍历**（后端基础）

### 1. 核心概念速览
MVCC（Multi-Version Concurrency Control）是 InnoDB 实现一致性非锁定读的核心机制：聚簇索引记录通过隐藏列 DB_TRX_ID、DB_ROLL_PTR 与 undo log 构成版本链，Read View 负责判定某个版本对当前事务是否可见，使快照读不申请行锁、不阻塞写，写操作也不阻塞快照读。它解决读写并发下的锁竞争与一致性视图问题，是 RC/RR 隔离级别、快照读/当前读语义、长事务与 undo purge 行为的底层基础。位置：位于事务引擎的并发控制层，与记录锁/间隙锁/next-key lock 共同构成隔离级别实现。本质不是保存多份完整行，而是用事务 ID 和 undo 版本链重建历史行版本，再做可见性判定；不掌握它就无法准确解释不可重复读、幻读、长事务导致 undo 膨胀、主从延迟与死锁排查。

### 2. 底层原理剖析
1）物理基础：InnoDB 聚簇索引行格式含隐藏列 DB_TRX_ID（最近修改该行的事务 ID）、DB_ROLL_PTR（指向 undo 中前版本记录的指针），无主键时还有 DB_ROW_ID。UPDATE 不覆盖旧版本，而是写入新记录并让 DB_ROLL_PTR 指向 undo 中的旧列值；undo 记录自身也含 trx_id 与 roll_ptr，从而形成按时间倒序的版本链。删除通过 delete_flag 标记，可见版本若标记删除则视为不存在。insert undo 主要服务回滚，update undo 参与 MVCC。

2）Read View 字段：m_ids 为创建瞬间活跃读写事务 ID 集合；min_trx_id 为其中最小事务 ID；max_trx_id 为下一个待分配事务 ID；creator_trx_id 为创建者事务 ID。Read View 的本质是事务活跃集合快照，不是行数据快照。

3）可见性判断伪代码：

    可见(trx_id)：
        if trx_id == creator_trx_id: return true   -- 自己的修改对自己可见
        if trx_id < min_trx_id: return true        -- Read View 创建前已提交
        if trx_id >= max_trx_id: return false      -- Read View 创建后才开启
        if trx_id in m_ids: return false           -- 创建时仍活跃，未提交
        return true                                -- 创建前已提交且不在活跃集合

4）版本链遍历：从聚簇索引中的最新记录开始，取 DB_TRX_ID 做可见性判断；不可见则沿 DB_ROLL_PTR 取 undo 中的前版本，重复判断，直到找到可见版本或链尾；若可见版本 delete_flag 为真则视为不存在。二级索引叶子节点不存 DB_TRX_ID/DB_ROLL_PTR，通常需回表到聚簇索引判断可见性。

5）Read View 生成时机：RC 下每条快照读 SELECT 生成新 Read View；RR 下事务内第一次快照读生成并复用，START TRANSACTION WITH CONSISTENT SNAPSHOT 则在事务启动时立即生成；当前读（UPDATE/DELETE/SELECT ... FOR UPDATE/LOCK IN SHARE MODE）读取最新已提交版本并加锁，不使用 Read View。

6）与前端不可变状态/闭包快照的异同：相同点是读操作绑定一个固定版本视图，后续写入不改变旧视图；不同点是前端快照通常复制或持有对象引用，MVCC 不复制整行，而是追加 undo 并通过 DB_ROLL_PTR 回溯，且可见性由 m_ids/min_trx_id/max_trx_id/creator_trx_id 判定，不是引用相等或单一版本号比较。

### 3. 基础代码与实战验证
```text
验证脚本（两个会话；A 为观察事务，B 为修改事务）：

    -- 初始数据
    CREATE TABLE t (id INT PRIMARY KEY, v INT) ENGINE=InnoDB;
    INSERT INTO t VALUES (1,10);

    -- 会话 A：RR
    SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
    START TRANSACTION;
    SELECT v FROM t WHERE id=1; -- 返回 10；第一次快照读生成 Read View：记录 m_ids、min_trx_id、max_trx_id
    -- 会话 B 执行：START TRANSACTION; UPDATE t SET v=20 WHERE id=1; COMMIT;
    -- 更新生成新版本：新记录 DB_TRX_ID=B，DB_ROLL_PTR 指向 undo 中 v=10 的旧版本
    SELECT v FROM t WHERE id=1; -- RR 复用第一次 Read View；B 的 trx_id 不可见，沿 DB_ROLL_PTR 回溯到旧版本，仍返回 10
    COMMIT;

    -- 会话 A：RC 对照
    SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
    START TRANSACTION;
    SELECT v FROM t WHERE id=1; -- 每条 SELECT 生成新 Read View；此时 B 已提交，返回 20
    -- 会话 B 执行：UPDATE t SET v=30 WHERE id=1; COMMIT;
    SELECT v FROM t WHERE id=1; -- 新 Read View 中 B 不再活跃，返回 30
    COMMIT;

    -- 当前读对照
    SELECT v FROM t WHERE id=1 FOR UPDATE; -- 当前读读最新已提交版本并加锁，不走 Read View；返回 30
```

### 4. 常见误区与进阶思考
误区 1：认为 RR 的 Read View 在 BEGIN 时生成。实际在事务内第一次快照读时生成；只有 START TRANSACTION WITH CONSISTENT SNAPSHOT 才在事务启动时立即生成。RC 则是每条快照读 SELECT 都生成新 Read View。

误区 2：认为 MVCC 让所有读都不加锁、RR 完全消除幻读。MVCC 只作用于快照读；UPDATE/DELETE/SELECT ... FOR UPDATE/LOCK IN SHARE MODE 是当前读，读取最新提交版本并依赖记录锁/间隙锁/next-key lock。RR 快照读可避免幻读，但当前读与快照读混用仍可能出现幻读语义。

思考题：RR 事务 A 在 T1 执行快照读生成 Read View；随后 B 插入 id=2 并提交；A 执行 UPDATE t SET v=v+1 WHERE id=2。该 UPDATE 影响 0 行还是 1 行？A 再执行 SELECT * FROM t WHERE id=2 是否可见？请用当前读、版本链、creator_trx_id 解释。关键提示：UPDATE 是当前读，能读到 B 已提交行；更新后该行 DB_TRX_ID 变为 A 自己，自身修改对 Read View 可见。
