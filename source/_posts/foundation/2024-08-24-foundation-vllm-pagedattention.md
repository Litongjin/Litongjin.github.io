---
title: "每日基础技术总结 · 2024-08-24 · 推理框架：vLLM 的 PagedAttention"
date: 2024-08-24 20:00:00
categories: [技术分享]
tags: ["技术分享", "AI / LLM 工程实战"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-08-24 · 推理框架：vLLM 的 PagedAttention

## 📚 今日主题

> **推理框架：vLLM 的 PagedAttention**（AI / LLM 工程实战）

### 1. 核心概念速览
PagedAttention 是 vLLM 的核心调度原语，旨在解决 LLM 推理中 KV Cache（键值缓存）的内存碎片化与低效利用问题。传统自回归生成模型在显存管理上复用操作系统的虚拟内存分页机制：将连续的 KV Cache 逻辑序列映射到非连续的物理块（Blocks）。它消除了内部碎片（Block 内未使用部分）和外部碎片（无法分配的离散空闲块），使 GPU 显存利用率接近理论极限，从而支持更大的 Batch Size 和更长的上下文窗口，是高吞吐 LLM 推理服务的基石。

### 2. 底层原理剖析
运行机制分为逻辑页表映射与物理块分配两个维度。
1. 逻辑结构：每个 Sequence Group 维护一个逻辑页表（Logical Page Table），数组索引对应 token ID，值为物理 Block ID。
2. 物理存储：KV Cache 被切分为固定大小的物理块（Physical Blocks），存储在 GPU 连续显存中。Block Size 通常为 16-32 个 tokens。
3. 分配策略：采用类似操作系统 Buddy System 或 First-Fit 的策略分配物理 Block。当 Sequence 增长时，仅在页表中追加新 Block 指针，无需移动数据。
4. 查询过程：Attention 计算时，根据逻辑页表索引定位物理 Block，读取对应 chunk 的 K/V 数据参与计算。
对比前端概念：这类似于 JavaScript 中的对象属性查找或哈希表映射，但层级更深。前端 TS/JS 接口定义静态类型约束，而 PagedAttention 的页表是动态运行时数据结构；若类比数据库，它相当于 InnoDB 的 B+Tree 叶子节点分页存储，通过元数据（页号）间接寻址，实现数据的逻辑连续性与物理分布性的解耦。

### 3. 基础代码与实战验证
```text
// 简化版 PagedAttention 逻辑页表操作伪代码
// BlockSize = 16, MaxSequenceLength = 1000

class PagedMemoryManager {
    // 物理块池：实际存储 KV Cache 的连续显存缓冲区
    private physicalBlocks: Array<Buffer>; 
    // 逻辑页表：每行代表一个 Sequence，每列代表该 Sequence 的 Token 所在的物理块ID
    private logicalPageTable: Map<SequenceId, Array<BlockId>>;

    constructor() {
        this.physicalBlocks = allocateGPUMemory(CHUNK_COUNT * BLOCK_SIZE);
        this.logicalPageTable = new Map();
    }

    // 核心方法：为新的 Sequence 初始化逻辑页表
    initSequence(seqId: number) {
        // 分配初始 Block ID 序列，此时尚未真正写入 KV 数据
        const initialBlockIds = allocateNewBlocks(Math.ceil(MAX_LEN / BLOCK_SIZE));
        this.logicalPageTable.set(seqId, initialBlockIds);
    }

    // 核心方法：Sequence 生成新 Token 时的扩展操作
    appendToken(seqId: number, blockIndex: number, kvData: Tensor) {
        // 1. 计算当前 Token 所属的物理块偏移量
        const globalBlockIdx = Math.floor(blockIndex / BLOCK_SIZE);
        const localOffset = (blockIndex % BLOCK_SIZE) * HEAD_DIM; // Head Dim per KV pair

        // 2. 从逻辑页表获取对应的物理块指针
        const physicalBlockId = this.logicalPageTable.get(seqId)[globalBlockIdx];

        // 3. 直接写入 GPU 显存指定偏移位置，无拷贝开销
        writeToGPU(physicalBlockId + localOffset, kvData);
    }

    // 核心方法：Attention 计算时的内存布局重组
    buildAttentionLayout(seqId: number) {
        // 将分散的物理块 ID 转换为 Attention Kernel 可处理的线性索引数组
        return this.logicalPageTable.get(seqId).map(id => id * BLOCK_SIZE);
    }
}
```

### 4. 常见误区与进阶思考
误区1：认为 PagedAttention 仅仅是一种数据压缩技术。实质它是一种内存管理机制，目的是消除碎片而非压缩数据体积，KV 数据本身并未减少，只是不再强制连续存储。
误区2：混淆 Prefix Caching 与 PagedAttention。PagedAttention 解决单序列内的内存碎片，Prefix Caching 解决多序列间重复前缀的 KV Cache 共享（类似 Git 的 commit 快照），两者正交且常配合使用。
深度思考题：在多模态大模型（如 LLaVA）场景中，Image Embedding 的 KV Cache 通常很长且一次性加载。如果 Image Embedding 占用大量连续 Block，而 Text Token 增量式插入，这种异构长度的 KV Cache 混合存储会如何影响 PagedAttention 的 Block 分配效率？请从 Block Size 粒度和内存碎片率角度推导潜在瓶颈。
