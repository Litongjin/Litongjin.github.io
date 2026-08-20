---
title: "每日基础技术总结 · 2026-08-05 · PostgreSQL 的 HOT（Heap-Only Tuple）与可见性映射 VM"
date: 2026-08-05 08:00:00
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-08-05 · PostgreSQL 的 HOT（Heap-Only Tuple）与可见性映射 VM

## 📚 今日主题

> **PostgreSQL 的 HOT（Heap-Only Tuple）与可见性映射 VM**（后端基础）

### 1. 核心概念速览
HOT（Heap-Only Tuple）是 PostgreSQL 在堆表上实现的一种元组更新优化机制，用于在索引键未变化时避免产生新的索引条目。其本质是：在一次 UPDATE 中，如果被修改的列不涉及任何索引列，则新版本元组直接放在原元组所在页面的空闲空间中，并通过在原元组头部设置 t_ctid 指向新版本，形成版本链；同时，在索引项中保留原元组的指针（TID），查询时通过索引找到旧版本再沿版本链找到最新版本。VM（Visibility Map）是每个表文件旁的一个附属结构（fork），以页为单位记录两个位：全可见位（all-visible）和全冻结位（all-frozen）。它解决的核心问题是：加速 VACUUM 的扫描范围（跳过全可见页）以及支持 Index-Only Scan（若页全可见则无需回表检查可见性）。在整个体系里，HOT 属于存储引擎与并发控制（MVCC）的交界，VM 属于存储管理与查询优化器的辅助结构。专业工程师必须掌握它们，因为它们直接决定写入放大、索引膨胀、Vacuum 负载和只读查询性能，是理解 PostgreSQL 性能调优、数据布局设计以及类似 LSM-Tree 架构对比的基石。

### 2. 底层原理剖析
HOT 的触发条件严格且机制精妙。设表有索引列 I，执行 UPDATE 时：1) 若新元组所有被更新的列均不属于任何索引列（即不改变索引键值），且 2) 新元组能放入与原元组同一个数据页（page）的空闲空间，则触发 HOT。此时新元组称为 HOT 链的后续版本。原元组的 t_ctid 字段指向新元组的位置（页号+偏移），新元组头部的 t_infomask2 会设置 HEAP_HOT_UPDATED，而旧元组设置 HEAP_ONLY_TUPLE。索引项保持不变，仍指向旧元组 TID；查询时通过索引找到旧元组，然后沿 t_ctid 链遍历，直到找到对当前事务可见的版本（需要检查 xmin/xmax 快照）。注意：如果索引列被更新，则必须在该索引中插入新条目，此时不能 HOT；如果新版本无法放入同一页，也不能 HOT。VM 的机制更为直白：每个堆页在 VM 中对应两个 bit——all-visible 表示该页中所有元组对所有活跃事务都可见（即该页没有需要清理的死元组，且所有元组的 xmin 均早于当前最小活跃事务 xmin 快照），all-frozen 表示该页所有元组均为冻结状态（xmin 小于 vacuum 阈值，无需再处理）。VACUUM 扫描时，若页面 all-visible 已置位，可跳过整个页面的清理；Index-Only Scan 时，若页面 all-visible 为真，则无需回表检查可见性，直接返回索引中的元组数据。VM 的更新是延迟的，由后台进程（如 bgwriter 或 vacuum）在适当时机设置/清除，因此存在短暂的不一致，但不会影响正确性，因为 VACUUM 和 Index-Only Scan 必须始终通过快照验证。前端类比：HOT 类似于前端中在对象属性不变时只更新其引用而避免触发依赖重新收集的优化（如 React 的 bailout），但本质上是物理存储层面的版本链复用；VM 类似于前端构建系统中的增量缓存——若模块没有变化则跳过构建（跳过页面），同时用于告知消费者“该模块的导出无需检查变更”（Index-Only Scan）。但两者都是存储引擎的物理层机制，与前端语言层面的抽象（如接口）完全不同。

### 3. 基础代码与实战验证
```text
-- 以下为验证 HOT 与 VM 的极简 SQL 实验步骤（需超级用户权限）。

-- 1. 创建测试表：包含一个索引列和一个非索引列
CREATE TABLE hot_test (
    id INT PRIMARY KEY,        -- 索引列
    val TEXT                    -- 非索引列
);

-- 2. 插入初始数据
INSERT INTO hot_test VALUES (1, 'a'), (2, 'b');

-- 3. 检查当前页面的可见性映射状态（初始应为全可见）
SELECT * FROM pg_visibility WHERE relname = 'hot_test';

-- 4. 执行不更新索引列的 UPDATE（触发 HOT）
BEGIN;
UPDATE hot_test SET val = 'c' WHERE id = 1;  -- id 未变，val 非索引列，若新版本同页则 HOT
-- 查看元组头部 t_ctid 和 HOT 标志
SELECT ctid, xmin, xmax, (t_infomask & 0x0002) AS is_hot_updated FROM hot_test;
-- 实际应通过扩展查看：SELECT lp_off, lp_flags, t_ctid, t_infomask2 FROM heap_page_items(get_raw_page('hot_test', 0));
COMMIT;

-- 5. 验证索引项未增加（使用 pg_stat_all_indexes 的 idx_scan 或直接检查索引页大小）
SELECT pg_relation_size('hot_test_pkey');  -- 与初始时对比，无变化则说明未新增索引条目

-- 6. 验证 VM 作用：强制 vacuum 后观察 VM 状态
VACUUM hot_test;
SELECT all_visible, all_frozen FROM pg_visibility WHERE relname = 'hot_test';
-- 若页面全部可见，则 Index-Only Scan 可跳过回表：
SET enable_seqscan = off;
EXPLAIN (ANALYZE) SELECT val FROM hot_test WHERE id = 1;  -- 计划中若使用 Index-Only Scan 且 Heap Fetches 为 0，则 VM 生效。

-- 注：真实 HOT 触发需满足同页空间充足。若更新导致新版本跨页，则不会触发 HOT。
-- 可通过增大填充因子控制同页空间：CREATE TABLE hot_test (id INT PRIMARY KEY, val TEXT) WITH (fillfactor = 70);
```

### 4. 常见误区与进阶思考
误区一：认为 HOT 减少了所有 UPDATE 的开销。实际上 HOT 仅适用于索引列未被更新的场景，且新版本必须与原版本同页。若表没有索引（堆组织表），则每次 UPDATE 天然没有索引条目更新，但 PostgreSQL 中 heap-only tuple 机制仍然存在，只是没有索引指向旧元组；但此时“HOT”的意义不大，因为无需维护索引。更常见的误区是：HOT 不保证消除版本链——当多次 HOT 更新且中间有并发事务占用空间时，版本链可能断裂（非 HOT 元组插入中间），导致后续访问链变长。

误区二：认为 VM 的 all-visible 位是精确的、实时一致的。VM 中的位是异步清除的，可能在事务提交后的一段时间内仍为 false，导致 VACUUM 和 Index-Only Scan 无法利用优化。这不影响正确性，但影响性能。反之，如果并发写入使得页面不再全可见，VM 位会延迟清除，可能在极短窗口内读到过期元组吗？实际上不会，因为任何查询都必须通过快照判断可见性，VM 只是“可跳过”的充分条件，不是必要条件。

思考题：在 HOT 更新后，如果旧元组被某个长事务（其快照早于更新事务）看到，而新元组对所有新事务可见，此时该页面是否可能被标记为 all-visible？如果不能，为什么？请从 MVCC 的可见性规则和 VM 的定义出发，解释该场景下 VACUUM 和 Index-Only Scan 的行为。
