---
title: "每日基础技术总结 · 2026-04-25 · LSM-Tree 的 Leveled Compaction 与写放大"
date: 2026-04-25 20:00:00
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-04-25 · LSM-Tree 的 Leveled Compaction 与写放大

## 📚 今日主题

> **LSM-Tree 的 Leveled Compaction 与写放大**（后端基础）

### 1. 核心概念速览
Leveled Compaction 是 LSM-Tree（Log-Structured Merge-Tree）中的一种核心合并策略，旨在通过分层级的数据重组消除重写放大（Rewrite Amplification），从而降低读放大并优化磁盘 I/O 吞吐。其本质是将内存中的 MemTable 持久化为不同大小的 SSTable 文件，并按层级组织：第 L0 层存储任意大小的文件，L1 至 Ln 层存储大小相近的文件。Compaction 过程仅在相邻层级间发生，将较小层级的所有文件与较大层级中重叠的数据合并，生成一个更大但有序的新文件替换旧文件。该机制解决的核心问题是传统 B+Tree 在随机写入场景下导致的全页重写开销，使 LSM-Tree 具备极高的写入吞吐量。在后端基础体系中，它是 RocksDB、Cassandra、HBase 等高性能 KV 存储或列式数据库的基石；对于专业工程师，掌握此概念是理解高并发写场景下的 I/O 瓶颈、延迟抖动及存储引擎底层性能调优的前提。

### 2. 底层原理剖析
Leveled Compaction 的运行逻辑基于‘空间换时间’与‘预排序’原则。其核心机制如下：
1. 层级结构约束：L0 层允许存在不同大小和数量的 SSTable，且键范围可重叠；Li 层（i > 0）内的所有 SSTable 大小近似相等，且键范围互不重叠，形成连续的区间。
2. 触发条件：当 L0 层 SSTable 数量超过阈值，或 Li 层达到满容量时，启动 Compaction。
3. 合并过程：选取 L0 中的一个 SSTable S_L0 和 Li 中与它键范围重叠的所有 SSTables {S_i1, S_i2, ...}。将这些文件中的所有记录读取出来，按 Key 排序后写入一个新的 SSTable S'_i (大小为各源文件大小之和)。随后删除源文件 S_L0 和 {S_ik}。
4. 迭代下沉：若新文件 S'_i 导致 Li 层容量超标，则继续将该新文件作为输入，与 Li+1 层重叠文件进行同样操作，直至数据稳定在某个层级。

与前端知识体系的对比：
类比 TypeScript 中的类型继承与接口实现。TS 接口（Interface）定义契约，类（Class）实现具体逻辑。LSM 的 MemTable 像是一个临时的、无序的 Map（类似 JS Object），快速写入；SSTable 则是持久化的、有序的结构。Leveled Compaction 类似于代码重构（Refactoring）：将分散、冗余、未优化的临时代码块（L0 小文件）逐步整合、整理成规范、高效、可复用的模块库（Ln 大文件）。B+Tree 的节点分裂像是不定长数组的动态扩容，而 LSM 的分层合并像是构建静态类型系统下的严格布局，虽然初始写入有额外的元数据维护成本（Compaction 开销），但读路径（Seek/Get）的时间复杂度被严格控制在 O(log N)，且消除了写放大。

### 3. 基础代码与实战验证
```text
// 伪代码展示 Leveled Compaction 的核心逻辑
// 假设 File 为 SSTable 结构，包含 key-range [min_key, max_key] 和 records []

class LevelDBStorage {
    levels = [[], [], []]; // L0, L1, L2...
    
    // 1. Write Path: MemTable flush to L0
    async function flushMemTableToL0(memTable) {
        let sstable = await memTable.toSortedSSTable();
        this.levels[0].push(sstable);
        // 检查是否触发 compaction
        if (this.levels[0].length > MAX_L0_FILES) {
            this.compactLevels(0);
        }
    }

    // 2. Compaction Logic
    async function compactLevels(level) {
        if (level < this.levels.length - 1) {
            // 获取 L0 中所有待合并的文件
            let sources = this.levels[level];
            if (!sources.length) return;
            
            // 找出下一级中所有与 sources 范围重叠的文件
            let targetFiles = [];
            for (let file of sources) {
                targetFiles.push(...getOverlappingFiles(this.levels[level + 1], file));
            }
            
            // 合并所有相关文件
            let mergedRecords = new PriorityQueue();
            for (let f of [...sources, ...targetFiles]) {
                f.records.forEach(r => mergedRecords.enqueue(r));
            }
            
            // 写入新的 SSTable 到下一级
            let newSSTable = await createSSTableFromPriorityQueue(mergedRecords);
            this.levels[level + 1].push(newSSTable);
            
            // 异步删除源文件
            deleteFilesAsync(sources); 
            deleteFilesAsync(targetFiles);
            
            // 递归检查下一层是否溢出（Write Amplification Chain）
            if (isLevelFull(level + 1)) {
                this.compactLevels(level + 1);
            }
        }
    }
    
    // 关键注释：Leveled 机制下，每个 Key 在任一时刻仅存在于一个非 L0 文件中。
    // 因此，单次写入最终最多参与 log_{Fanout}(Size_of_DB) 次磁盘重写。
    // 这就是所谓的 'Amplification Factor'，通常远小于 Universal Compaction 的 N 倍。
```

### 4. 常见误区与进阶思考
['误区一：混淆‘写放大’（Write Amplification）与‘CPU 开销’。Leveled Compaction 的写放大主要指物理磁盘上的额外 IO 次数（即为了写入 1MB 数据，实际执行了大于 1MB 的磁盘读写），而非 CPU 排序计算开销。许多开发者误以为 CPU 计算密集就是写放大，实际上在 SSD 时代，I/O 带宽和寿命才是瓶颈，CPU 加速比往往不是首要矛盾。', '误区二：认为 Leveled Compaction 完全消除了乱序。Leveled 仅保证了 Li 层（i>0）内部有序且无重叠，L0 层始终是无序且允许重叠的。因此，即使使用 Leveled Compaction，查询 L0 层仍需要多级查找或位图定位，并非所有写入都能立即获得 O(log N) 的读取效率，冷数据需要经历完整的 Compaction 链才能获得最佳读性能。']
