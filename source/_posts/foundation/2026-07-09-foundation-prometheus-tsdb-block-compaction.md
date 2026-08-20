---
title: "每日基础技术总结 · 2026-07-09 · Prometheus TSDB 的 block 与 compaction 及写入放大"
date: 2026-07-09 08:00:00
categories: [技术分享]
tags: ["技术分享", "DevOps 与云原生"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-09 · Prometheus TSDB 的 block 与 compaction 及写入放大

## 📚 今日主题

> **Prometheus TSDB 的 block 与 compaction 及写入放大**（DevOps 与云原生）

### 1. 核心概念速览
Prometheus TSDB 的 block 是存储引擎中不可变的、自包含的时间序列数据分片，每个 block 包含按时间排序的样本数据、索引文件（倒排索引）和元数据。compaction 是将多个小 block 合并为更大 block 的过程，其本质是归并排序并去重/删除旧数据，以控制 block 数量、压缩索引和减少内存开销。写入放大（write amplification）指在 compaction 过程中，实际写入磁盘的数据量远大于原始写入的数据量，因为每个样本可能被多次读取、重写。该机制解决的是长时运行下的存储碎片化和查询效率退化问题，位于 Prometheus 存储层核心，是理解 TSDB 性能特征和容量规划的基础。专业工程师必须掌握，因为任何生产级时序数据库的写入吞吐、磁盘占用和查询延迟都直接由 block 调度与 compaction 策略决定。

### 2. 底层原理剖析
TSDB 底层基于 WAL + 内存 chunk 与磁盘 block 两级结构。新数据先写入 WAL，然后周期性内存中 chunk 被刷盘成 block。每个 block 内部包含 chunk 文件（样本压缩）、index 文件（倒排索引：标签到 series，series 到 chunk）和 meta.json。compaction 流程本质是归并：给定一组时间范围重叠或相邻的 block，读取每个 block 中的 series 样本，按时间戳归并，应用删除墓碑（tombstone）和样本保留策略，再写出新 block，更新 meta.json 并原子替换旧 block。关键点：样本在合并过程中被重新序列化压缩，因此存在写入放大。写入放大系数 = 总写盘字节数 / 新摄入样本字节数。由于每 2 小时产生一个 block，长期运行产生大量 block，查询需要合并多个 block 的结果，因此需要 compaction 减少 block 数量。对比前端：block 类似不可变数据（如 React 的不可变 props），compaction 类似 V8 的 Mark-Compact 垃圾回收——都是将零散数据整理为连续存储以提升访问效率；但 V8 压缩阶段是移动对象指针，而 TSDB compaction 是重新编码样本，且用户可控制触发时机。写入放大类似于 GC 中复制算法造成的移动成本，但 TSDB 的放大系数取决于数据重叠度和保留策略，而非存活率。伪代码：function compact(blocks): out = new Block(); for each series in merge(blocks): for each sample in series.samples: out.write(sample); ... 其中每个 sample 可能从源 block 读入再写出，重复多次。

### 3. 基础代码与实战验证
```text
class Block:
    def __init__(self, start, end):
        self.start = start
        self.end = end
        self.samples = {}  # sid -> list[(ts, val)]

    def add(self, sid, ts, val):
        self.samples.setdefault(sid, []).append((ts, val))

def compact(blocks, start, end):
    merged = {}
    original_count = 0  # 原始摄入样本数（所有block中样本总数，不去重）
    for b in blocks:
        for sid, samples in b.samples.items():
            original_count += len(samples)
            merged.setdefault(sid, []).extend(samples)  # 直接合并列表
    out = Block(start, end)
    write_count = 0
    for sid, samples in merged.items():
        samples.sort(key=lambda x: x[0])
        prev_ts = None
        for ts, val in samples:
            if ts == prev_ts:
                continue
            prev_ts = ts
            out.add(sid, ts, val)
            write_count += 1
    # 写入放大 = (原始摄入写盘 + compaction写盘) / 原始摄入写盘
    # 原始摄入每个样本写一次，compaction每个去重后样本写一次
    amp = (original_count + write_count) / original_count
    return out, amp

# 验证：两个时间重叠的block
b1 = Block(0, 10)
b1.add('s1', 5, 1.0)
b1.add('s1', 6, 2.0)
b2 = Block(5, 15)
b2.add('s1', 6, 3.0)  # 与b1中ts=6重复，compaction去重（保留前者）
b2.add('s1', 7, 4.0)

out, amp = compact([b1, b2], 0, 15)
print('compacted samples:', len(out.samples['s1']))
print('write amplification factor:', amp)
# 输出: compacted samples: 3, write amplification factor: 1.75 (7/4)
```

### 4. 常见误区与进阶思考
误区1：认为compaction能完全消除写入放大。实际只能通过调整block大小和压缩算法降低，无法消除，因为合并必然需要重新读取和写回。
误区2：认为compaction是纯后台操作，不影响查询。实际上compaction期间会占用CPU/IO，且查询需要访问正在合并的block，可能通过内存映射和文件锁，导致查询延迟增加或暂时失败。
思考题：如果两个block时间范围不重叠，compaction是否还会产生写入放大？为什么？更深入：在compaction过程中，如何保证查询的一致性？提示：使用atomic rename和只读block。
