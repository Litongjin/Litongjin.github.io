---
title: "每日基础技术总结 · 2026-11-03 · MySQL InnoDB页分裂与页合并对碎片率的具体影响"
date: 2026-11-03 07:01:23
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-03 · MySQL InnoDB页分裂与页合并对碎片率的具体影响

## 📚 今日主题

> **MySQL InnoDB页分裂与页合并对碎片率的具体影响**（后端基础）

### 1. 核心概念速览
核心概念速览
InnoDB 以 16KB 页作为 B+Tree 节点、缓冲池缓存和磁盘 IO 的基本单位；聚簇索引叶子页存储行记录，非叶页存储键值与子页号。页分裂是插入路径上的结构维护操作：目标页无足够空间容纳新记录时，分配新页，按分裂点迁移记录，维护页内记录链表与 Page Directory，并在父页插入新页指针；父页满时递归分裂。页合并是删除/更新/purge 后的收缩操作：当页的有效记录占用低于 MERGE_THRESHOLD（默认 50%）时，尝试与相邻页合并，重组记录并把释放页放入表空间 free list。
碎片率必须分三层看：页内碎片率约为 1 - DATA_SIZE/16384，反映页内空闲空间与记录空洞；页间碎片体现为 B+Tree 索引页填充率低、页数膨胀、树高增加；表空间碎片体现为 DATA_FREE 表示的空闲 extent 字节数。页分裂的具体影响是产生非满页、降低平均填充率、增加索引页数、缓冲池 miss、随机 IO 与写放大；页合并的具体影响是提高页填充率、减少索引页数、释放可复用 extent，但独立表空间文件不会自动收缩，且合并后若重新插入可能再次分裂。
该知识点位于存储引擎物理层，直接连接 B+Tree、Buffer Pool、Redo/Undo、Purge、表空间与 extent 管理。专业工程师必须掌握它，因为主键设计、批量写入、历史数据归档、容量规划、慢查询诊断、OPTIMIZE/Online DDL 决策和云数据库成本都受其影响。与前端已有体系对比：JS 数组中间插入会触发连续内存搬移与容量扩容；InnoDB 页分裂是页级记录迁移加父页递归更新，并伴随 redo/undo 与脏页；页合并类似 GC compact，但它是局部、阈值触发、非全局停止的在线操作。

### 2. 底层原理剖析
底层原理剖析
插入路径：1）从根页经非叶页定位目标叶子页，持有相应 latch；2）在页内通过 Page Directory 槽二分定位，记录本身按主键有序链表组织；3）若页内连续空闲空间不足，进入分裂；4）分裂时从 free list 或新 extent 分配页，选择分裂点，把一部分记录复制/移动到新页，更新前后页指针、Page Directory、页头统计与父页子指针；5）父页无空间则递归分裂；6）所有结构变更写 redo，页变为脏页，由 checkpoint/刷脏落盘。分裂点由记录数、插入位置和平衡策略决定，常见中间分裂会使两页都非满；顺序右端追加时主要在最右页分裂，历史页可保持高填充；随机主键插入会频繁命中不同页，长期形成较低平均填充率。
删除路径：1）记录先 delete-mark；2）purge 线程后续物理删除，记录空间进入页内 free list，形成页内空洞；3）当页使用率低于 MERGE_THRESHOLD，InnoDB 尝试与左或右兄弟页合并；4）若两页记录可放入一页，则重组记录、更新父页指针、释放一页到表空间 free list；5）释放页可被后续插入复用，但文件大小不自动缩小。合并阈值过低会积累页间碎片；过高会增加合并频率与后续再分裂概率，形成分裂-合并抖动。
碎片度量可写成：页内碎片率 ≈ 1 - DATA_SIZE/16384；平均填充率 = ΣDATA_SIZE/(N×16384)；DATA_FREE 是表空间空闲 extent 字节数，不等同于页内碎片。页分裂主要恶化页间填充率与索引页数；页合并主要改善页间填充率并释放 extent，但 DATA_FREE 可能上升，因为页被释放为空闲 extent 而文件未收缩。
与前端对比：JS 数组扩容是容量翻倍加连续内存复制，中间 splice 是 O(n) 元素搬移；InnoDB 页分裂是 O(log n) 定位后做页内记录链表重组、页间链接和父页递归，成本体现在 latch、redo、脏页与随机 IO。浏览器 GC compact 是全局或代际压缩；InnoDB 页合并是索引页局部、由删除/purge 和阈值触发的在线收缩。

### 3. 基础代码与实战验证
```text
基础代码与实战验证
以下 SQL 用顺序主键与随机 UUID 主键对比页分裂计数、缓冲页填充率和 DATA_FREE，并验证删除后的页合并与表空间碎片。

-- 开启并重置页分裂/合并计数器，COUNT 为累计值，分阶段重置才能隔离实验
SET GLOBAL innodb_monitor_enable = 'index_page_splits';
SET GLOBAL innodb_monitor_enable = 'index_page_merge_attempts';
SET GLOBAL innodb_monitor_enable = 'index_page_merges';
SET GLOBAL innodb_monitor_reset_all = 'index_page_splits';
SET GLOBAL innodb_monitor_reset_all = 'index_page_merge_attempts';
SET GLOBAL innodb_monitor_reset_all = 'index_page_merges';
SET SESSION cte_max_recursion_depth = 100000;

CREATE DATABASE IF NOT EXISTS frag_lab;
USE frag_lab;
DROP TABLE IF EXISTS t_seq, t_rand;
-- 顺序主键：插入点集中在最右页，分裂次数应显著少，历史页填充率高
CREATE TABLE t_seq (id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY, pad CHAR(200) NOT NULL DEFAULT '') ENGINE=InnoDB;
-- 随机主键：UUID 使插入点分散到全树，易触发多页分裂并降低平均填充率
CREATE TABLE t_rand (id CHAR(36) PRIMARY KEY, pad CHAR(200) NOT NULL DEFAULT '') ENGINE=InnoDB;

-- 顺序插入 20000 行
INSERT INTO t_seq(pad)
WITH RECURSIVE s(n) AS (SELECT 1 UNION ALL SELECT n+1 FROM s WHERE n<20000)
SELECT REPEAT('a',200) FROM s;

-- 查看顺序主键阶段的页分裂计数
SELECT NAME, COUNT FROM information_schema.INNODB_METRICS
WHERE NAME IN ('index_page_splits','index_page_merge_attempts','index_page_merges');

-- 重置计数器后做随机 UUID 插入
SET GLOBAL innodb_monitor_reset_all = 'index_page_splits';
SET GLOBAL innodb_monitor_reset_all = 'index_page_merge_attempts';
SET GLOBAL innodb_monitor_reset_all = 'index_page_merges';
INSERT INTO t_rand(id,pad)
WITH RECURSIVE s(n) AS (SELECT 1 UNION ALL SELECT n+1 FROM s WHERE n<20000)
SELECT UUID(), REPEAT('b',200) FROM s;

-- 对比随机主键阶段的页分裂计数；通常 index_page_splits 明显更高
SELECT NAME, COUNT FROM information_schema.INNODB_METRICS
WHERE NAME IN ('index_page_splits','index_page_merge_attempts','index_page_merges');

-- 全表扫描把索引页载入缓冲池，再按页统计 DATA_SIZE 近似填充率
SELECT COUNT(*) FROM t_seq;
SELECT COUNT(*) FROM t_rand;
SELECT INDEX_NAME, COUNT(*) AS pages, ROUND(AVG(DATA_SIZE)/16384,4) AS avg_fill
FROM information_schema.INNODB_BUFFER_PAGE
WHERE PAGE_TYPE='INDEX' AND (INDEX_NAME LIKE '%t_seq%' OR INDEX_NAME LIKE '%t_rand%')
GROUP BY INDEX_NAME;

-- 删除随机表大部分记录，delete-mark 后由 purge 物理删除，页使用率下降触发合并尝试
SET GLOBAL innodb_monitor_reset_all = 'index_page_merge_attempts';
SET GLOBAL innodb_monitor_reset_all = 'index_page_merges';
DELETE FROM t_rand LIMIT 18000;
DO SLEEP(3); -- 给 purge 线程推进时间，使页内空洞可被合并逻辑观察到
SELECT NAME, COUNT FROM information_schema.INNODB_METRICS
WHERE NAME IN ('index_page_merge_attempts','index_page_merges');

-- DATA_FREE 是表空间空闲 extent 字节数；页合并后可能上升，因为页被释放但文件未收缩
SHOW TABLE STATUS LIKE 't_rand';
-- 重建表可回收表空间碎片，但会带来 IO/锁成本，不能简单等同于修复页分裂
OPTIMIZE TABLE t_rand;
SHOW TABLE STATUS LIKE 't_rand';
```

### 4. 常见误区与进阶思考
常见误区与进阶思考
误区一：把 DATA_FREE 大直接等同于页分裂严重或必须 OPTIMIZE。DATA_FREE 是表空间空闲 extent 字节数，受删除、页合并、extent 分配策略和文件是否收缩共同影响；页分裂更直接的表现是索引页数增加、平均填充率下降、缓冲池命中率降低和写放大。页合并释放页后 DATA_FREE 反而可能上升，但索引实际占用页数下降，查询和缓存效率可能改善。
误区二：认为页合并阈值越高越好，或页合并没有代价。提高 MERGE_THRESHOLD 会让更多低填充页参与合并，短期减少页数，但合并本身需要 latch、记录重组和 redo；若业务随后在同一范围重新插入，会再次分裂，形成分裂-合并抖动并放大写 IO。阈值必须匹配插入/删除模式，随机主键导致的分裂问题应优先通过主键设计、分区、归档策略解决，而不是只调阈值。
思考题：一张高写入时序表同时存在历史数据批量删除和近期数据随机回填，若将 MERGE_THRESHOLD 从默认 50 提高到 90，请分析 index_page_splits、index_page_merge_attempts、index_page_merges、平均页填充率、Buffer Pool 命中率和写放大将如何变化；为什么可能出现合并次数上升但碎片率没有持续下降，甚至插入延迟变差？
