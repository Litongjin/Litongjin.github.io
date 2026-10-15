---
title: "每日基础技术总结 · 2026-10-16 · JVM G1 GC  Mixed Garbage Collection 的混合收集判断逻辑"
date: 2026-10-16 07:01:08
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-16 · JVM G1 GC  Mixed Garbage Collection 的混合收集判断逻辑

## 📚 今日主题

> **JVM G1 GC  Mixed Garbage Collection 的混合收集判断逻辑**（后端基础）

### 1. 核心概念速览
G1 Mixed GC 是 G1 收集器在并发标记完成后执行的、同时回收 Young Generation 与部分老年代 Region 的增量式收集模式。其判断逻辑由四个核心参数构成：G1HeapWastePercent 决定是否进入 Mixed GC；G1MixedGCLiveThresholdPercent 决定哪些 Old Region 有资格进入候选集；G1OldCSetRegionThresholdPercent 与停顿目标决定每次 Mixed GC 实际回收的 Old Region 数量；G1MixedGCCountTarget 决定整个 Mixed 序列的预算次数。本质是：基于并发标记得到的 Region 级存活数据，以低停顿为目标、按回收收益对老年代空间做多次、小批量的子集回收，从而替代 Full GC。它在 JVM GC 体系中处于 Young GC 与 Full GC 之间，是 G1 实现可预期停顿时间的关键机制，也是后端工程师调优延迟与避免 Full GC 的核心知识。

### 2. 底层原理剖析
前置：G1 堆被划分为相同大小的 Region，逻辑上仍分 Young/Old。IHOP（默认 45%）触发并发标记；标记完成后，每个 Old Region 都有 liveness 数据。\n\n1. 候选集构建：遍历所有 Old Region，若其 live_percent ≤ G1MixedGCLiveThresholdPercent（默认 85%，即垃圾占比 ≥ 15%），则加入候选集 C。\n2. 触发判断：计算 C 的可回收字节总和 reclaimable_bytes = Σ(region.used - region.live)。若 reclaimable_bytes / total_heap_size < G1HeapWastePercent（默认 5%），放弃进入 Mixed GC，继续纯 Young GC；否则进入第 3 步。\n3. 执行 Mixed GC 序列：\n   - 每轮 Mixed GC 的 CSet = 全部 Young Region + 一部分候选 Old Region。\n   - 候选 Old Region 按 GC efficiency = reclaimable_bytes / predicted_collection_time 降序选择。\n   - 约束条件：Old Region 数量 ≤ G1OldCSetRegionThresholdPercent（默认 10%）× 总 Region 数；预测停顿 ≤ GCPauseTimeMillis（默认 200ms）。\n   - 若候选为空或剩余可回收空间占比 < G1HeapWastePercent，或已执行次数 ≥ G1MixedGCCountTarget（默认 8），停止 Mixed GC，回到 Young GC。\n   - 否则继续下一轮 Mixed GC。\n\n与前端已有概念的对比：V8/Orinoco 的 GC 采用分代 + Major GC 的整代标记-清除/整理，触发主要依赖堆增长与内存压力启发式；其并发标记/增量清扫虽降低停顿，但不存在 Region 级候选集、回收收益过滤和次数目标。G1 的 Mixed GC 本质是并发标记后的离散化、收益驱动的增量回收，将“一次大型老年代回收”拆成多次带约束的小型回收，这是前端 V8 当前 GC 机制所不具备的。

### 3. 基础代码与实战验证
```text
验证方式：固定小堆，制造老年代垃圾，使用显式并发标记触发混合收集，并通过 GC 日志观察 CSet 选择。\n\nJava 代码：\npublic class G1MixedGCTest {\n    public static void main(String[] args) throws Exception {\n        java.util.List<byte[]> list = new java.util.ArrayList<>();\n        // 分配 128 个 1MB 对象，在 256MB 堆下促使对象晋升到老年代\n        for (int i = 0; i < 128; i++) {\n            list.add(new byte[1024 * 1024]);\n        }\n        // 释放前 64 个，前半段变为老年代垃圾；后半段保持存活\n        for (int i = 0; i < 64; i++) {\n            list.set(i, null);\n        }\n        // System.gc() 配合 ExplicitGCInvokesConcurrent，触发并发标记而非 Full GC\n        System.gc();\n        Thread.sleep(500);\n        // 继续分配，产生 Young GC，使满足阈值的场景进入 Mixed GC 序列\n        for (int i = 0; i < 64; i++) {\n            list.add(new byte[1024 * 1024]);\n        }\n    }\n}\n\n运行参数：\njava -Xms256m -Xmx256m \\\n     -XX:+UseG1GC \\\n     -XX:+ExplicitGCInvokesConcurrent \\\n     -XX:G1HeapWastePercent=5 \\\n     -XX:G1MixedGCLiveThresholdPercent=85 \\\n     -XX:G1OldCSetRegionThresholdPercent=10 \\\n     -XX:G1MixedGCCountTarget=8 \\\n     -Xlog:gc+cset=trace:file=gc.log \\\n     G1MixedGCTest\n\n日志观察重点：并发标记结束后，若老年代候选 Region 的可回收字节总和占总堆 ≥ 5%，则日志中会出现 Mixed GC 的 CSet 同时包含 Young 和 Old Region；Old Region 数量受 10% Region 上限约束；若 ≤ 5% 则不会进入 Mixed GC，只有 Young GC。
```

### 4. 常见误区与进阶思考
误区一：把 IHOP 触发并发标记当作 Mixed GC 触发。实际上 IHOP（默认 45%）只决定何时开始并发标记；并发标记完成后要再做一次“可回收空间 / 堆总大小 ≥ G1HeapWastePercent（默认 5%）”的判断，不满足则跳过 Mixed GC。\n误区二：以为 Mixed GC 一旦开始就会回收所有候选老年代 Region。实际它是一系列受预算约束的增量回收：每轮只选回收收益最高的一部分 Region，且 Old Region 数量最多占总 Region 10%，最长执行 G1MixedGCCountTarget 次；剩余候选会在后续 Mixed GC 或下个标记周期处理。\n思考题：若将 G1MixedGCLiveThresholdPercent 调整为 100、G1HeapWastePercent 调整为 0，并保持 G1OldCSetRegionThresholdPercent 与停顿目标不变，会出现什么现象？请从候选集规模、回收效率、停顿时间与晋升失败四个维度推导。这本质是考察你能否区分“有资格进入候选集”和“真正值得回收”两个不同阶段。
