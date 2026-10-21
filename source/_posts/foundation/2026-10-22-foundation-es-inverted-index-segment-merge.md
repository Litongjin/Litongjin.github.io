---
title: "每日基础技术总结 · 2026-10-22 · Elasticsearch倒排索引的建立过程与Segment合并策略"
date: 2026-10-22 07:05:08
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-22 · Elasticsearch倒排索引的建立过程与Segment合并策略

## 📚 今日主题

> **Elasticsearch倒排索引的建立过程与Segment合并策略**（后端基础）

### 1. 核心概念速览
## Elasticsearch 倒排索引的建立过程与 Segment 合并策略：核心概念速览

**倒排索引（Inverted Index）** 是搜索引擎的核心数据结构，其本质是建立 **从词项（Term）到文档集合（Posting List）** 的映射，与正向索引（文档→词项）形成对偶关系。在 Lucene/Elasticsearch 中，倒排索引并非单一结构，而是由 **词典（Term Dictionary，通常以 FST 压缩存储）、倒排表（Posting List，存储 docID、词频 TF、位置 Position、偏移 Offset）、以及用于加速跳读的 Skip List** 三部分构成。

**Segment 合并策略（Merge Policy）** 是解决 LSM-Tree 架构下索引碎片化问题的核心机制。ES 采用 **不可变段（Immutable Segment）** 设计，写入操作生成新的 Segment 文件（不可变保证无锁并发与缓存友好），后台通过 **分层合并（Tiered Merge Policy，默认策略）** 将多个小 Segment 归并为大 Segment，以控制 **Segment 数量**（直接影响查询时需要遍历的倒排表数量）与 **磁盘空间回收**（清理被标记删除的文档）。

在计算机体系结构中，这是 **LSM-Tree（Log-Structured Merge-Tree）** 思想在搜索引擎的具体实现：通过顺序写（Append-only）牺牲部分读性能换取高吞吐写入，再通过后台合并（Compaction）重构数据局部性以优化读。与 B+ Tree 的随机写原地更新不同，LSM 将随机 I/O 转化为顺序 I/O，这对处理高并发写入与近实时搜索（NRT）至关重要。

专业工程师必须掌握此机制，因为它决定了索引的 **写入吞吐、查询延迟、磁盘膨胀率** 三者的权衡边界。任何索引优化（如调整 `refresh_interval`、配置 `merge.policy`、处理 `indexing_pressure`）本质都是在调优这两个过程的参数。

## 底层原理剖析

### 1. 倒排索引的建立流程（Index Chain）

文档写入至可搜索状态需经历以下不可变阶段：

**阶段一：Analysis（分析器流水线）**
原始文本经 Character Filter（字符级预处理）→ Tokenizer（分词，生成 Token Stream）→ Token Filter（标准化，如 lowercase、stop words、stemming）。输出为 **Term Vector**，包含每个词项的元数据（position、offset、payload）。

**阶段二：内存缓冲与 Translog**
文档首先写入 **Indexing Buffer**（默认占堆内存 10%）的内存倒排索引（In-memory Buffer），同时追加到 **Translog（WAL）** 确保持久性。此时文档对搜索不可见（未生成文件系统缓存中的 Segment）。

**阶段三：Refresh（刷新）**
触发条件（默认 `index.refresh_interval: 1s` 或 buffer 满）：
1. 内存 buffer 中的数据被 **冻结** 为新的 Segment（生成 `.cfs` 复合文件或分离的 `.fdt/.fdx/.tim` 等文件）
2. 文件写入 OS Cache（Page Cache）并打开文件句柄（`fsync` 由后续 Flush 保证）
3. 清空内存 buffer
4. 新 Segment 加入 IndexReader 视图，搜索可见

**阶段四：Flush（落盘）**
默认每 30 分钟或 Translog 达阈值（`index.translog.flush_threshold_size`）：执行 `fsync` 将 OS Cache 中的 Segment 强制写入磁盘，并清空 Translog（因为数据已持久化到 Segment 文件）。

**关键机制**：Segment 一旦生成即 **不可变（Immutable）**。更新操作实则是 **标记删除（Del 文件）+ 新文档插入**，删除的文档在物理层保留直到 Segment 合并时被真正回收。

### 2. Segment 合并策略（TieredMergePolicy）

默认的 **分层合并（Tiered）** 策略区别于 **日志合并（LogMergePolicy）**，其选择合并集（Merge Set）的逻辑如下：

1. **排序**：按 Segment 大小降序排列
2. **评估**：计算每个 Segment 的删除比例（`numDeletedDocs / maxDoc`），仅当删除比例超过阈值（默认 `expunge_deletes_allowed: 10%`）或符合分层规则时纳入合并候选
3. **分层选择**：寻找一组连续 Segment，使得它们的总大小不超过当前层的 **MaxMergedSegmentBytes**（默认 5GB），且层数不超过 **SegmentsPerTier**（默认 10）
4. **执行**：将选中的 Segments 合并为新的大 Segment，旧文件在确认无 Reader 引用后物理删除

**与前端构建工具的对比**：
- **Segment 生成** 类似于 Webpack 的 **Code Splitting** 产物：每次构建生成新的 Chunk（Segment），不可变且通过 Hash（Segment 版本号）标识，便于 CDN 缓存（OS Cache 缓存 Segment 文件）。
- **Segment 合并** 类似于 **Tree Shaking + Scope Hoisting**：将多个小 Chunk 合并为大 Chunk，消除重复代码（删除文档），优化加载时的网络请求数（减少查询时需要打开的倒排表文件数）。
- **Translog** 类似于 Service Worker 的 **Background Sync**：先写入持久化日志（WAL），确保崩溃恢复后未落盘的变更可重放，再异步同步到主存储（Segment 文件）。

**与 B+ Tree 的关键差异**：B+ Tree 原地更新导致随机写和页分裂，而 LSM/Segment 机制将更新转化为 **顺序写 + 后台 Compaction**，牺牲读放大（Read Amplification，需查询多个 Segment 后合并结果）换取写吞吐和 SSD 寿命。

## 基础代码与实战验证

以下伪代码精确描述倒排索引建立与 Segment 合并的内存/磁盘交互逻辑：

// 倒排索引构建流程（伪代码）
function buildInvertedIndex(documents) {
    // 阶段1：内存构建（对应 ES 的 Indexing Buffer）
    const memoryBuffer = new InvertedIndexBuffer(); // 哈希表：Term → PostingList
    const translog = new WriteAheadLog("translog.ckp"); // 预写日志

    for (const doc of documents) {
        // Analysis：生成 Term Vector
        const terms = analyze(doc.body);

        // 写入内存倒排索引（无锁，因线程隔离或线程安全队列）
        for (const term of terms) {
            // PostingList 结构：链表或跳表，存储 [docID, freq, [pos1, pos2...], [offset...]]
            memoryBuffer.add(term, doc.id, term.positions, term.offsets);
        }

        // 写入 Translog（顺序 I/O，保证崩溃恢复）
        translog.append({ op: "index", doc, terms });
    }

    // 阶段2：Refresh（生成不可变 Segment）
    function refresh() {
        // 将内存 buffer 序列化为磁盘格式的 Segment 文件（仍位于 OS Cache）
        const segment = memoryBuffer.serializeToSegment(); // 生成 .tim, .tip, .doc 等文件
        segment.setImmutable(true); // 关键：标记不可变，允许并发读无锁

        // 打开文件句柄，使其对搜索可见（OS Page Cache 层）
        segment.openForSearch();

        // 清空内存 buffer，准备接收新写入
        memoryBuffer.clear();

        // 注意：此时未执行 fsync，数据仅存在于 Page Cache
        return segment;
    }

    // 阶段3：Flush（持久化）
    function flush() {
        for (const segment of unflushedSegments) {
            segment.fsync(); // 强制写入磁盘，确保持久性
        }
        translog.truncate(); // 截断旧日志，因为数据已安全落盘
    }
}

// Segment 合并策略核心逻辑（TieredMergePolicy 简化版）
function selectMergeCandidates(segments) {
    // 按大小降序排序
    const sorted = segments.sort((a, b) => b.sizeInBytes - a.sizeInBytes);

    const maxMergedSegment = 5 * 1024 * 1024 * 1024; // 5GB 默认上限
    const maxSegmentsPerTier = 10;

    const candidates = [];

    // 贪心选择：寻找总大小不超过阈值且包含删除文档的 Segment 集合
    for (let i = 0; i < sorted.length; i++) {
        const current = sorted[i];

        // 检查删除比例（Expunge Deletes）
        const deleteRatio = current.numDeletedDocs / current.maxDoc;
        if (deleteRatio < 0.10 && !isTooManySegments()) continue; // 默认 10% 删除比例阈值

        // 尝试构建合并集（Merge Set）
        const mergeSet = [current];
        let totalSize = current.sizeInBytes;

        // 向后合并较小的 Segment（类似滑动窗口）
        for (let j = i + 1; j < sorted.length && mergeSet.length < maxSegmentsPerTier; j++) {
            if (totalSize + sorted[j].sizeInBytes > maxMergedSegment) break;
            mergeSet.push(sorted[j]);
            totalSize += sorted[j].sizeInBytes;
        }

        // 如果找到有效合并集（至少 2 个，或单个但需清理删除）
        if (mergeSet.length > 1 || deleteRatio > 0.10) {
            candidates.push(mergeSet);
            i += mergeSet.length - 1; // 跳过已选中的 Segment
        }
    }

    // 执行合并：将多个 Segment 的 Posting List 归并排序，物理删除标记为已删除的 docID
    return candidates;
}

// 合并执行细节
function executeMerge(mergeSet) {
    const newSegment = new SegmentWriter("merged_" + generateId());

    // 多路归并（Multi-way Merge）：同时迭代多个 Segment 的词典
    const iterators = mergeSet.map(seg => seg.getTermIterator());

    while (hasNext(iterators)) {
        const minTerm = findMinTerm(iterators); // 最小词项优先

        // 收集该词项在所有 Segment 中的 Posting List
        const postings = iterators
            .filter(it => it.term() === minTerm)
            .flatMap(it => it.postings());

        // 合并并过滤删除文档（检查 Del 文件中的位图）
        const validPostings = postings.filter(p => !isDeleted(p.docID, mergeSet.delFiles));

        newSegment.writeTerm(minTerm, validPostings);
    }

    // 原子性提交：先写新 Segment，再更新元数据（segments_N 文件），最后删除旧 Segment
    newSegment.commit();
    updateSegmentsMetadata(newSegment);
    deleteObsoleteSegments(mergeSet); // 确认无 Reader 引用后物理删除
}

## 常见误区与进阶思考

**误区一：认为 Refresh 会执行 fsync 落盘**
许多工程师混淆 `refresh_interval`（默认 1s）与 Flush 机制。Refresh 仅将内存 buffer 转为 **OS Cache 中的 Segment 文件**（此时可通过 `GET /index/_search` 查询到），但并未执行 `fsync`，宕机仍可能丢失最近 1 秒的数据。真正的持久化依赖 Flush（默认 30 分钟或 Translog 满），后者通过 `fsync` 将 Segment 强制写入磁盘并截断 Translog。

**误区二：忽视 Force Merge 对磁盘空间的瞬时冲击**
执行 `POST /index/_forcemerge?max_num_segments=1` 时，ES 需要 **原索引大小 + 新合并后大小** 的磁盘空间（最多 2 倍），因为合并是写新文件、删旧文件的过程。在磁盘水位（`cluster.routing.allocation.disk.watermark.high` 默认 90%）接近时触发 Force Merge 会导致集群只读。此外，Force Merge 是 **同步阻塞** 操作，会占用大量 CPU 和 I/O，应在业务低峰期执行。

**进阶思考题**
在 TieredMergePolicy 下，如果某索引的写入速率极高（如日志场景，每秒生成 50 个 10MB 的小 Segment），而删除率极低（<1%），默认的合并策略可能导致 **Segment 数量爆炸**（超过 `index.max_merge_at_once` 限制）或 **合并带宽不足**（合并速度跟不上生成速度），进而触发 `indexing_pressure` 拒绝写入。

**问题**：在此场景下，应如何调整 `index.merge.policy.segments_per_tier`（默认 10）与 `index.merge.policy.max_merge_at_once`（默认 10）的参数组合，以平衡 **写放大（Write Amplification）**、**读放大（Read Amplification）** 与 **空间放大（Space Amplification）**？请从 LSM-Tree 的层间大小比例（Size Ratio）理论推导最优配置，而非仅给出经验值。
