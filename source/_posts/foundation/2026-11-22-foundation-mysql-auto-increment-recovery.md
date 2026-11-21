---
title: "每日基础技术总结 · 2026-11-22 · MySQL自增主键连续性与重启后最大值的恢复逻辑"
date: 2026-11-22 07:02:29
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-22 · MySQL自增主键连续性与重启后最大值的恢复逻辑

## 📚 今日主题

> **MySQL自增主键连续性与重启后最大值的恢复逻辑**（后端基础）

### 1. 核心概念速览
本质：InnoDB 的 AUTO_INCREMENT 不是『列的属性』，而是每表一份、引擎私有的表级单调递增计数器（dict_table_t 中的 autoinc 字段），它与三样东西刻意解耦——与行数据解耦（发号发生在写入 B+ 树之前，回滚/唯一键冲突后值不回收）、与事务解耦（发号动作不登记 undo log、不受两阶段锁约束）、与磁盘解耦（MySQL 5.7 及以前仅存内存）。它给出的契约只有两条：值唯一 + 分配序列只向前；『连续』从未被承诺。5.7：计数器是派生状态，重启后以 MAX(auto_col)+1 从聚集索引最右叶重建——删除过最大行后重启即发生值回退与复用，空表连同建表指定的 AUTO_INCREMENT 初值一并回落到 1。8.0：计数器升级为持久化的源状态——每次变化写入 redo log、每次 checkpoint 固化到引擎私有系统表，重启以『字典持久值 + redo 重放』恢复崩溃前水位。它解决的问题是『高并发原子发号（O(1) CAS）+ 聚簇索引顺序追加写（避免页分裂）+ 崩溃一致性』三者的工程折中。在体系中的位置：这是『进程易失状态与磁盘持久状态对齐』的最小教科书案例，与 Redis AOF、Kafka offset checkpoint、Raft log index、Snowflake 时钟回拨同属崩溃恢复/发号器问题域，也是 statement 复制一致性与分库分表全局 ID 设计的前置知识。业务主键、幂等去重键、AI 数据管道的 dedup 表都建立在其真实语义之上，误把『连续/不复用』当契约会在存储层埋雷。

### 2. 底层原理剖析
一、状态模型：两层状态刻意分离
① 行数据：磁盘上的聚集索引 B+ 树，受事务协议（undo/redo/binlog）管辖。
② 发号器：表级 autoinc 计数器，只做原子递增，不参与事务协议。
写入路径（所有版本共通）：
insert_row(row):
  if row.id IS NULL:
      id = ATOMIC_FETCH_ADD(&table.autoinc, 1)   # 表级元数据上的原子递增，永不递减
      row.id = id                                 # 关键：这一步不写 undo，不属于任何事务
  write (id, row) into clustered B+tree           # 这一步才进入事务协议
推论：ROLLBACK 回滚的是第二步，第一步的计数器位移不可逆 → 永久空洞；唯一键冲突、INSERT...ON DUPLICATE KEY UPDATE 走 UPDATE 分支同理：预留的号已被计数器消耗。

二、5.7 崩溃恢复：派生态重建（值可回退）
on_restart(table):
  counter = (聚集索引最右叶一次顺序扫描得 MAX(id)) + 1
  空表 → counter = 1（建表指定的 AUTO_INCREMENT=N 初值同时丢失）
设计取向：发号水位不持久化（省掉每次插入的额外 IO），重启后由磁盘数据完全重算。代价：所有『曾分配但已不在磁盘』的值（被删的最大行、回滚/冲突的空洞）在重启后全部回炉复用。

三、8.0 崩溃恢复：WAL 重放（水位不丢）
on_restart(table):
  counter = 引擎私有系统表中的持久化值              # 每次 checkpoint 固化一次
  for 该表 autoinc 的每条 redo 记录:               # 覆盖 checkpoint 后到崩溃前的全部增长
      counter = max(counter, 记录值)                # 只增不减地折叠
设计取向：每次计数器变化追加一条小 redo 记录（顺序写、组提交摊薄，吞吐代价可忽略），水位因此获得崩溃一致性。注意 8.0 修复的只是『重启回退』，回滚/冲突造成的逻辑空洞依旧存在。

四、为什么回滚必须不回收
① 回收是非幂等操作：需引入全局 free-list 与跨事务协调，摧毁 CAS 发号 O(1) 无争用路径；
② 复用小 id = 聚簇索引中段插入 = 页分裂，直接违背『自增主键换顺序追加写』的设计初衷；
③ statement 复制要求发号序列在从库可确定性重放。
结论：以可控空洞换取并发度、物理有序性与可复制性——并发系统的经典取舍。

五、AUTO-INC 锁三模式（与复制格式是耦合决策对）
mode 0 traditional：任何插入持表级 AUTO-INC 锁至语句结束，发号全序，吞吐最差。
mode 1 consecutive（5.7 默认）：行数已知的 simple insert 仅用 mutex 一次性预留连续块；行数未知的 bulk（INSERT...SELECT / LOAD DATA）退回 mode 0 锁，且预留按 1→2→4→2^k 翻倍增长，块未用满即为空洞。
mode 2 interleaved（8.0 默认）：永不持 AUTO-INC 锁，多语句交错发号，只保唯一不保连续；与 statement 复制天然不兼容（从库重放发号序列必然漂移）——所以 8.0 同步把默认 binlog 格式改为 ROW（行事件携带显式 id，从库不再重新发号）。lock mode 与复制格式必须成对理解。

六、与前端已有知识体系的映射
- JS 的 ++id 在单线程事件循环下天然原子串行；InnoDB 处于多线程，必须用 autoinc mutex / AUTO-INC 锁实现同等原子性，并砍掉『递减/回收』这个非幂等分支。单线程直觉在多线程存储引擎中失效，是理解本知识点的思维转换点。
- 5.7 计数器 ≈ 组件里生成 key 的 useRef 内存计数器：进程内派生态，刷新即丢，由持久数据（DOM/接口数据）重算；8.0 ≈ 计数器写入 localStorage 并叠加事件日志回放——Redux 时间旅行的 reducer replay 与 redo replay 完全同构：append-only 日志 + 折叠函数推导当前状态。
- 前端惯用 crypto.randomUUID() ≈ 数据库用 UUID 做主键：随机聚簇键使插入位置随机、频繁页分裂；自增整型保证单调追加到最右页。这是『InnoDB 偏好自增主键』的存储层答案，也同构于 React 列表 key 用稳定 id 而非数组索引的动因：索引会被复用，正如 5.7 重启后的值复用。

### 3. 基础代码与实战验证
```text
-- 文件：verify_autoinc.sql（mysql 客户端直接执行；5.7 与 8.0 对照跑效果最佳）
-- 验证目标：① 发号即消耗（回滚/冲突不回收） ② 重启后 5.7 回退 / 8.0 持久 ③ bulk 的 2^k 预留

SELECT @@version AS ver, @@innodb_autoinc_lock_mode AS lock_mode;
-- 8.0 默认 lock_mode=2、binlog=ROW；5.7 默认 lock_mode=1、binlog=STATEMENT

-- ========== 实验一：事务回滚不回收自增值 ==========
DROP TABLE IF EXISTS t;
CREATE TABLE t (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,  -- 聚集索引键；发号器是表级元数据，与此列解耦
  v  INT NOT NULL
) ENGINE = InnoDB;

BEGIN;
INSERT INTO t(v) VALUES (1);  -- 原子递增 → 分配 id=1；该分配不写 undo log
ROLLBACK;                     -- 行数据被 undo 回滚，计数器不回退（只向前语义）
INSERT INTO t(v) VALUES (2);  -- 得到 id=3：位置 2 成为永久空洞
SELECT * FROM t;              -- 结果 (3,2)：id ≠ 行数，连续性被证伪

-- ========== 实验二：唯一键冲突 / ON DUP 同样消耗发号 ==========
DROP TABLE IF EXISTS u;
CREATE TABLE u (
  id INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
  uk INT NOT NULL UNIQUE,      -- 唯一索引使『发号先于冲突检测』的消耗可见
  v  INT NOT NULL DEFAULT 0
) ENGINE = InnoDB;

INSERT INTO u(uk) VALUES (10);  -- id=1
INSERT INTO u(uk) VALUES (10);  -- Duplicate entry 报错，但 id=2 已被计数器消耗
INSERT INTO u(uk) VALUES (20);  -- id=3，跳过 2
INSERT INTO u(uk) VALUES (10) ON DUPLICATE KEY UPDATE v = v + 1;  -- 走 UPDATE 分支，simple-insert 仍预留 id=4
INSERT INTO u(uk) VALUES (30);  -- id=5
SELECT * FROM u;                -- 1,3,5：三处空洞的三个不同来源
SHOW CREATE TABLE u;            -- AUTO_INCREMENT=6：表级发号水位，与行内数据分离

-- ========== 实验三（核心）：重启后发号水位的恢复逻辑 ==========
DELETE FROM u WHERE id = 5;     -- 删掉最大行 → 数据层 MAX(id)=3，磁盘上已无 4/5
SHOW CREATE TABLE u;            -- 8.0：AUTO_INCREMENT 仍为 6（水位不受 DML 删除影响）
-- >>> 在操作系统层执行重启：systemctl restart mysqld <<<
SHOW CREATE TABLE u;
-- MySQL 5.7：水位仅存内存 → 重启后重算为 MAX(id)+1=4 → 后续 INSERT 复用 4、5（值回退/复用）
-- MySQL 8.0 ：水位已随每次变化写入 redo log 并 checkpoint 固化 → 重启重放恢复为 6 → 不复用
INSERT INTO u(uk) VALUES (40);  -- 5.7 得到 id=4；8.0 得到 id=6 —— 两个版本的行为分叉点
SELECT * FROM u;
-- 5.7 附带现象：DELETE 清空表后重启，水位回落 1，建表时 AUTO_INCREMENT=N 的初值一并丢失；
-- （区分：TRUNCATE 是 DDL，两个版本都会立即重置计数器，与重启无关）
-- 8.0 上述初值均已持久化保留。

-- ========== 实验四：bulk 插入的 2^k 翻倍预留（mode 1） ==========
SET SESSION innodb_autoinc_lock_mode = 1;
DROP TABLE IF EXISTS b;
CREATE TABLE b (id INT NOT NULL AUTO_INCREMENT PRIMARY KEY, v INT NOT NULL) ENGINE = InnoDB;

INSERT INTO b(v) VALUES (1),(2),(3),(4),(5);  -- simple insert：行数已知，语句开始一次性预留连续 5 个 → id 1..5 连续
INSERT INTO b(v) SELECT v FROM b;             -- 行数未知的 bulk：预留 1→2→4... 翻倍增长，块未用满即空洞
INSERT INTO b(v) SELECT v FROM b;
SHOW CREATE TABLE b;                          -- AUTO_INCREMENT 明显大于 COUNT(*)：预留空洞的直接证据
-- 若切回默认 mode 2（8.0）重跑，空洞来源变为『多语句交错发号』——模式不同，结论一致：连续性无保证
```

### 4. 常见误区与进阶思考
误区一：把『连续』和『重启不回退』当成 AUTO_INCREMENT 的契约。它只承诺唯一性与只向前的分配顺序；回滚、唯一键冲突、ON DUPLICATE KEY UPDATE、bulk 的 2^k 预留、mode 2 的交错发号都会制造空洞；5.7 下删除最大行后重启还会复用已消耗过的值。把 id 连号当业务语义（订单连号、漏号告警、用 MAX(id) 推断总量）是将存储引擎实现细节上浮为业务契约，正确做法是业务序号独立发号。工程红线：5.7 + ROW 复制下，主库重启回退后复用旧值而从库仍持有原值，会出现重复主键或主从数据分叉——发号器不在一致性协议内是该问题的结构根源。
误区二：认为『8.0 修复了回退 = 计数器与数据强一致 / 永无空洞』。redo 持久化只解决崩溃导致的水位丢失，回滚与冲突造成的逻辑空洞在 8.0 依旧存在。另一面，5.7 的 MAX+1 重建并非 bug，而是『不想为每次发号付 IO』约束下的理性设计（派生状态由磁盘重算）；8.0 是在 redo 顺序写 + 组提交的成本被证明可忽略后才翻转的取舍——同一问题、两种成本模型，理解取舍比记住结论重要。
进阶思考题：事务 T1 插入一行获得 id=100 后 ROLLBACK，随后实例崩溃重启，再插入一行——5.7 与 8.0 下新 id 各是多少？进一步：若崩溃前主库已把一条『分配了 id=100 的语句』以 ROW 格式写入 binlog 并送达从库，5.7 重启回退后主库继续复用 100 会发生什么？由此论证：8.0 把发号水位写入 redo 的真正动机，是让发号器纳入崩溃一致性协议（与 redo/binlog 的原子边界对齐），而不只是消除表面上的数值回退现象。
