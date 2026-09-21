---
title: "每日基础技术总结 · 2024-05-26 · 注意力变体：MQA/GQA 与长上下文"
date: 2024-05-26 20:00:00
categories: [技术分享]
tags: ["技术分享", "AI / LLM 工程实战"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-05-26 · 注意力变体：MQA/GQA 与长上下文

## 📚 今日主题

> **注意力变体：MQA/GQA 与长上下文**（AI / LLM 工程实战）

### 1. 核心概念速览
MQA (Multi-Query Attention) 和 GQA (Grouped-Query Attention) 是针对 Transformer 解码阶段生成效率的量化压缩变体。传统 MHA 中，每个查询头拥有独立的键(K)和值(V)向量组，导致 KV Cache 体积随头数线性增长，形成显存带宽瓶颈。MQA 将所有查询头共享同一组 K/V，GQA 则将查询头划分为若干组，每组共享 K/V。本质是通过牺牲少量表示能力的并行性冗余，换取巨大的 IO 吞吐增益。长上下文 (Long Context) 则是上述优化得以在超长序列上落地的基础，涉及从 O(N^2) 到 O(N) 或更低复杂度的算法重构（如 RoPE, Streaming/Sliding Window），旨在突破标准注意力机制在处理海量 Token 时的内存与计算双重限制。专业工程师需掌握其以理解 LLM 推理引擎的内存布局优化、KV Cache 管理及分布式通信瓶颈。

### 2. 底层原理剖析
1. 机制解构：
- MHA: Q_i = W_q * x, K_i = W_k * x, V_i = W_v * x。Softmax(QK^T)V。每个 Head i 有独立参数。
- MQA: Q_i 独立，但 K_all 共享一个投影矩阵 W_K^global，V_all 共享 W_V^global。注意：Q 仍保持维度分离以维持多路并发，但 K/V 的访存量降至 1/N head。
- GQA: H 个 Query Heads 分为 G 组 (G < H, G > 1)。每组内的 Q 独立，但该组内所有 Q 共享该组的单一 K/V 对。平衡了带宽压力与模型精度。

2. 前端对比映射：
- MHA 类似 React 中多个组件各自维护完整的 State 对象，虽然逻辑隔离但数据副本冗余。
- MQA/GQA 类似引入了 'Shared Store' 或 'Context API'。多个组件（Heads）读取同一个不可变的 Store（Global KV），写入时通过不同的 projection (Q weights) 转换后只读消费。这减少了内存拷贝（Memory Copy）和总线传输量，类似于前端中避免将大对象深拷贝传递，而是通过引用指针传递，配合 Immutable 结构保证安全的同时提升性能。

3. 长上下文核心：
- 位置编码：RoPE (Rotary Positional Embeddings) 将绝对位置信息融入旋转矩阵，允许模型外推至训练窗口外的长度，无需重新训练整个 embedding 层。
- 注意力稀疏化：FlashAttention 等算子利用 Tiling 技术，只在 SRAM 中计算部分 Block 的 Softmax，再反传梯度或激活值，彻底打破 HBM (High Bandwidth Memory) 的读写墙。

### 3. 基础代码与实战验证
```text
// 伪代码展示 GQA 的内存布局与计算差异
// 假设: num_heads = 32, head_dim = 64, kv_head_groups = 4 (GQA)

def compute_attention_gqa(query_states, key_states_proj, value_states_proj):
    # query_states shape: [batch, seq_len, 32, 64]
    # 注意：key_states 和 value_states 在此处已聚合为每组分一份
    # key_states_merged shape: [batch, seq_len, 4, 64] 
    # value_states_merged shape: [batch, seq_len, 4, 64]
    
    # 1. 展开 Keys 和 Values 以匹配 Query Heads
    # 将 4 份 Key 复制 8 次 (32 / 4)，模拟广播效应，但实际硬件通过 Pointer 跳转实现而非物理复制
    keys_broadcasted = expand_to_match(query_states.shape[2], key_states_merged) 
    values_broadcasted = expand_to_match(query_states.shape[2], value_states_merged)
    
    # 2. 标准点积注意力 (利用 FlashAttention/Tiling 底层优化)
    # attn_weights = softmax(Q @ K.T / sqrt(d))
    scores = einsum('bqhd,bkhd->bhqk', query_states, keys_broadcasted)
    attn_probs = softmax(scores, dim=-1)
    
    # 3. 加权求和
    # output = attn_probs @ V
    context = einsum('bhqk,bkhd->bqhd', attn_probs, values_broadcasted)
    
    return context

# 关键差异点：
# MHA: keys_broadcasted 是 [batch, seq_len, 32, 64]，占用大量 HBM 带宽
# GQA: 物理存储仅 4 份 K/V，expand 操作多为零拷贝视图(view)或寄存器复用，显著降低 IO
```

### 4. 常见误区与进阶思考
['误区一：认为 MQA/GQA 会严重损害模型能力。实际上，现代预训练表明，只要 Group 数量合理（如 GQA 的 8-16 组），精度损失微乎其微，甚至因正则化效应略有提升，主要瓶颈在于推理侧的显存带宽而非训练侧的参数容量。', "误区二：混淆 '长上下文' 与 '简单的 Sequence Extension'。增加训练长度不等于支持长推理。若无 RoPE 缩放因子调整、无 FlashAttention 2+ 的算子支持、无 KV Cache 的滑动窗口策略，单纯增加 Layer 会导致 OOM (Out of Memory) 或注意力权重坍缩。"]
