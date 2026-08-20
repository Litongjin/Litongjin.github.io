---
title: "每日基础技术总结 · 2026-08-05 · LSM-Tree 的 Compaction 策略：Size-Tiered 与 Leveled 的写放大/读放大权衡"
date: 2026-08-05 08:00:00
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-08-05 · LSM-Tree 的 Compaction 策略：Size-Tiered 与 Leveled 的写放大/读放大权衡

## 📚 今日主题

> **LSM-Tree 的 Compaction 策略：Size-Tiered 与 Leveled 的写放大/读放大权衡**（后端基础）

### 1. 核心概念速览
LSM-Tree（Log-Structured Merge-Tree）是一种面向写优化的存储结构，核心机制是将随机写转化为顺序写：先写入内存中的 Memtable，达到阈值后刷成不可变的有序 SSTable 落盘，再通过后台 Compaction 逐步合并这些 SSTable 以维持整体有序性。Compaction 策略决定了合并的时机与方式，直接决定了系统的写放大（Write Amplification）、读放大（Read Amplification）和空间放大（Space Amplification）。Size-Tiered Compaction（STCS）以表大小为分组依据，将相近大小的多个表合并成一个更大的表；Leveled Compaction（LCS）将数据划分为多个层级，每层容量按倍数增长，逐层下推合并。STCS 写放大低、读放大和空间放大高；LCS 读放大低、空间放大低，但写放大高。该知识点属于存储引擎的核心基础，是 LevelDB/RocksDB/Cassandra 等系统性能调优的必知内容。专业工程师掌握它，能够理解数据库写入吞吐与查询延迟背后的根本权衡，而不是停留在 API 层面。

### 2. 底层原理剖析
一、Size-Tiered Compaction 机制：所有 SSTable 按大小分组（通常大小是 2 的幂）。当同一大小组的表数量达到阈值 T（典型为 4），取其中 T 个表做多路归并，生成一个大小乘以 T 的新表，放入更大的组。该过程持续进行，直到没有组超过阈值。STCS 的特点是合并粒度大、触发条件宽松，因此每条数据被重写的次数约为 log_T(N)，写放大较低。但读取时需要检查所有未合并的表，且表间 key 范围高度重叠，因此读放大高；同时由于存在大量临时大表，空间放大也高。
二、Leveled Compaction 机制：维护 L 层，第 i 层容量约为第 i-1 层的 T 倍（如 10）。数据从 level0 开始，level0 可以有多表，但其他层每层只有一个有序表（或少量不重叠的表）。当某一层数据总大小超过容量上限，后台将该层所有数据与下一层数据做归并，结果写入下一层，并清空本层。这保证了除 level0 外，每层内 key 不重叠，因此读取最多检查 level0 的所有表加每层一个表，读放大约为 L。但每次合并都会重写下一层的大量数据，且级联合并可能发生，所以写放大约为 O(T * log_T(N))，明显高于 STCS。
三、本质权衡：写放大源于数据被重复重写，读放大源于查询需要扫描的 SSTable 数量。STCS 通过减少合并次数降低写放大，但允许表间重叠；LCS 通过强制分层有序降低读放大，但牺牲写放大。
四、与前端概念的对比：前端中不可变更新与可变更新的权衡与之类似。Redux 的 reducer 返回新 state 类似 LCS：每次操作产生新的有序结构，读路径简单可预测，但内存/GC 开销高；直接修改 state 类似 STCS：写入快，但后续需要同步和整理，读取时需考虑多个版本。区别在于 LSM 的 Compaction 是异步后台任务，而前端状态更新通常是同步的；且 LSM 关注磁盘 IO 放大，前端关注内存与渲染性能。

### 3. 基础代码与实战验证
```text
function merge(a, b) {
  let i = 0, j = 0, out = [];
  while (i < a.length && j < b.length) {
    if (a[i] <= b[j]) out.push(a[i++]);
    else out.push(b[j++]);
  }
  out.push(...a.slice(i), ...b.slice(j));
  return out;
}

// Size-Tiered：同尺寸表两两合并
function stcsCompaction(tables, stat) {
  let groups = new Map();
  for (let t of tables) {
    let s = t.length;
    if (!groups.has(s)) groups.set(s, []);
    groups.get(s).push(t);
  }
  let changed;
  do {
    changed = false;
    for (let [size, list] of groups) {
      while (list.length >= 2) {
        changed = true;
        let a = list.shift(), b = list.shift();
        let m = merge(a, b);
        stat.writeAmp += m.length; // 合并重写 m.length 条数据
        let ns = size * 2;
        if (!groups.has(ns)) groups.set(ns, []);
        groups.get(ns).push(m);
      }
    }
  } while (changed);
  return [...groups.values()].flat();
}

// Leveled：当第 i 层大小超过容量，与第 i+1 层归并
function leveledCompaction(levels, maxSizes, stat) {
  for (let i = 0; i < levels.length - 1; i++) {
    if (levels[i].length > maxSizes[i]) {
      let m = merge(levels[i], levels[i+1]);
      stat.writeAmp += m.length; // 合并重写数据
      levels[i] = [];
      levels[i+1] = m;
    }
  }
}

// 验证 STCS：4 个大小为 2 的表
let stat = { writeAmp: 0 };
let tables = [[1,2],[3,4],[5,6],[7,8]];
let finalTables = stcsCompaction(tables, stat);
console.log('STCS 写放大', stat.writeAmp, '最终表数', finalTables.length);

// 验证 LCS：level0 有 8 条数据，level1 为空
stat = { writeAmp: 0 };
let levels = [[1,2,3,4,5,6,7,8], []];
let maxSizes = [2, Infinity];
leveledCompaction(levels, maxSizes, stat);
console.log('LCS 写放大', stat.writeAmp, '非空层数', levels.filter(l => l.length > 0).length);
```

### 4. 常见误区与进阶思考
误区一：认为 LCS 一定优于 STCS。实际上没有绝对优劣，STCS 适合写多读少、容忍高读放大的场景（如时序数据）；LCS 适合读多写少、对读延迟敏感的场景（如用户记录查询）。选择必须基于工作负载模型。
误区二：混淆写放大与写入延迟。Compaction 是异步后台执行，写放大高不一定导致单次写入变慢，但会持续消耗 IO 带宽，降低整体吞吐；同时读放大也受布隆过滤器、缓存等影响，不能只看 SSTable 数量。
思考题：在 LCS 中，假设每层容量是上一层的 T 倍，一条数据从 level0 写入并最终到达最深层，估算其被重写的最多次数，并解释为什么 LCS 的写放大通常高于 STCS。这需要你真正理解级联合并的传播过程，而不是背诵结论。
