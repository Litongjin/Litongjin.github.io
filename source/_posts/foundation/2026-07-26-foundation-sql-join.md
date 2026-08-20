---
title: "每日基础技术总结 · 2026-07-26 · SQL 中 Join 的嵌套循环、哈希连接与排序合并连接选择"
date: 2026-07-26 08:00:00
categories: [技术分享]
tags: ["技术分享", "架构与设计"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-26 · SQL 中 Join 的嵌套循环、哈希连接与排序合并连接选择

## 📚 今日主题

> **SQL 中 Join 的嵌套循环、哈希连接与排序合并连接选择**（架构与设计）

### 1. 核心概念速览
Join 是关系代数中的二元运算符，语义上是笛卡尔积经连接谓词过滤后的子集。SQL 是声明式语言，不指定物理实现；优化器在逻辑执行计划之上选择具体连接算法。三种基础算法分别是 Nested Loop Join、Hash Join 和 Sort-Merge Join。其本质是在时间/空间/IO 之间做折中：Nested Loop 用 CPU 迭代换取最小内存；Hash Join 用内存哈希表换取线性扫描；Sort-Merge 用排序代价换取顺序 IO 与范围匹配能力。解决的问题是：在给定表大小、索引状态、可用内存、数据分布和谓词类型（等值/非等值）下，选择代价最小的匹配策略。它在整个计算机体系中的位置属于数据库查询执行引擎的核心物理算子，直接影响 OLTP/OLAP 的响应时间。专业工程师必须掌握它，因为上层 ORM、缓存、索引优化都无法绕过执行计划；错误的选择会导致吞吐量呈指数级下降。

### 2. 底层原理剖析
嵌套循环连接（NLJ）：以外表为驱动，对每一行通过扫描或索引访问内表。伪代码：
for each outer_row:
  for each inner_row matching predicate via index or full scan: output(outer_row, inner_row)
无索引时时间复杂度 O(N*M)，有唯一索引时降为 O(N*logM)（B+树查找），有非唯一索引时 O(N*(logM+K))，K 是平均匹配数。它不要求连接键可排序或等值，适合小驱动表 + 大内表且内表有索引的场景。

哈希连接（HJ）：将连接分为两个阶段。构建阶段：选择较小的表（build input）扫描并计算连接键的哈希值，将行存入内存哈希表；探测阶段：扫描较大的表（probe input），对每行计算哈希值定位桶，再在桶内进行精确的键值比较（处理哈希冲突）。伪代码：
build: for r in build_table: h=hash(r.key); hash_table[h].append(r)
probe: for s in probe_table: h=hash(s.key); for r in hash_table[h]: if r.key==s.key: output(r,s)
平均时间复杂度 O(N+M)，但依赖哈希函数均匀性和内存容量。当哈希表超过 work_mem 时需要分区分批落盘，退化为多轮递归哈希。仅支持等值连接。

排序合并连接（SMJ）：分两步。第一步对两表按连接键排序（若索引已提供有序性则省去）；第二步使用双指针归并。伪代码：
sort left by key; sort right by key; i=0; j=0; while i<len(left) and j<len(right): if left[i].key==right[j].key: 输出该行与右组所有匹配行，i++ ; else if left[i].key<right[j].key: i++ ; else: j++
排序代价 O(NlogN+MlogM)，归并部分 O(N+M)，但在多对多重复键下需要回溯扫描，最坏退化为 O(N*M)（若所有键相等）。支持等值及范围谓词（>,<,>=,<=）。

优化器选择原则：基于统计信息（行数、NDV、分布直方图、内存参数）计算各算法的 CPU、IO、内存代价。通常：小表驱动大表且内表有索引→NLJ；等值连接且无索引或内存充足→HJ；非等值连接或输入已有序→SMJ。

与前端概念对比：NLJ 对应前端代码中的双重 for 循环，简单但 O(N*M)；HJ 对应使用 JavaScript Map/Set 预建索引，查找 O(1)；SMJ 对应归并排序中的合并阶段，但需要先排序。这类似于 TypeScript 接口是编译期结构约束而 Java 接口是运行期多态契约，不同算法也有不同的适用范围和前置条件。前端工程师常以循环直觉处理关联，而数据库必须利用索引和排序的物理特性来降低复杂度。

### 3. 基础代码与实战验证
```text
以下代码用纯 JavaScript 模拟三种连接算法，可在 Node.js 中直接运行。

const users = [
  { id: 1, name: 'Alice' },
  { id: 2, name: 'Bob' },
  { id: 3, name: 'Carol' }
];
const orders = [
  { userId: 2, item: 'A' },
  { userId: 3, item: 'B' },
  { userId: 2, item: 'C' },
  { userId: 1, item: 'D' }
];

// 1. Nested Loop Join：外层驱动，内层全扫
function nestedLoop(outer, inner) {
  const result = [];
  for (const o of outer) {                    // 外层：驱动表每行
    for (const i of inner) {                  // 内层：被驱动表每行（无索引则全表扫）
      if (o.id === i.userId) result.push({ ...o, ...i }); // 匹配谓词
    }
  }
  return result;
}

// 2. Hash Join：小表构建哈希表，大表探测
function hashJoin(build, probe) {
  const map = new Map();                      // 哈希桶
  for (const b of build) {                    // 构建阶段：扫描 build 输入
    if (!map.has(b.id)) map.set(b.id, []);    // 按连接键哈希
    map.get(b.id).push(b);                    // 同键行挂到同一桶
  }
  const result = [];
  for (const p of probe) {                    // 探测阶段：扫描 probe 输入
    const bucket = map.get(p.userId);         // O(1) 定位桶
    if (bucket) {
      // 桶内精确比较（数据库需额外校验哈希冲突）
      for (const b of bucket) result.push({ ...b, ...p });
    }
  }
  return result;
}

// 3. Sort-Merge Join：先排序，后归并
function sortMerge(left, right) {
  const l = [...left].sort((a,b) => a.id - b.id);       // 排序阶段（数据库可用索引）
  const r = [...right].sort((a,b) => a.userId - b.userId);
  const result = [];
  let i = 0, j = 0;
  while (i < l.length && j < r.length) {
    if (l[i].id === r[j].userId) {
      // 固定左行，扫描右组中所有匹配（处理一对多）
      let k = j;
      while (k < r.length && r[k].userId === l[i].id) {
        result.push({ ...l[i], ...r[k] });
        k++;
      }
      i++;
    } else if (l[i].id < r[j].userId) {
      i++; // 左键较小，前进左指针
    } else {
      j++; // 右键较小，前进右指针
    }
  }
  return result;
}

console.log(nestedLoop(users, orders));
console.log(hashJoin(users, orders));
console.log(sortMerge(users, orders));

执行结果集完全一致（顺序不同），证明了三种算法在逻辑上等价；实际数据库中由优化器根据物理代价选择其一。
```

### 4. 常见误区与进阶思考
常见误区 1：认为连接算法选择只取决于表大小。实际上，即使外表很大，如果内表连接键有高选择性的索引，NLJ 可能比 HJ 更快，因为它避免了对内表全表扫描和哈希构建的额外 IO/CPU。同样，若数据已按连接键有序，SMJ 可以免排序，成本远低于估算。

常见误区 2：认为 Hash Join 是等值连接的最优解，总能打败其他算法。实际上，当哈希表无法放入内存时，HJ 需要分区溢出到磁盘，产生额外的写读 IO；若连接键数据严重倾斜，某个桶极大，探测阶段退化为长链扫描，性能可能劣于 SMJ 或 NLJ。优化器必须结合统计信息和内存限制。

思考题：两个表各 1000 万行，连接键均为整数且分布严重倾斜——1% 的键值占了 90% 的行。在内存有限且无索引的情况下，三种算法分别会如何退化？请从哈希桶冲突、排序后重复键组大小、嵌套循环的内外表扫描次数三个角度分析，并说明为何优化器可能最终选择 Sort-Merge Join 而不是 Hash Join。
