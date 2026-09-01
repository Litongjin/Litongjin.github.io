---
title: "每日基础技术总结 · 2026-09-02 · Transformer 架构简述"
date: 2026-09-02 07:01:49
categories: [技术分享]
tags: ["技术分享", "AI 开发基础（LLM & Agent）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-02 · Transformer 架构简述

## 📚 今日主题

> **Transformer 架构简述**（AI 开发基础（LLM & Agent））

### 1. 核心概念速览
Transformer 是一种基于纯注意力机制（Attention Mechanism）的序列到序列（Seq2Seq）架构，由 Vaswani et al. 在 2017 年提出。其本质是：通过自注意力（Self-Attention）对输入序列中所有位置两两计算相关性权重，并据此加权聚合全局信息，从而直接建模任意两个 token 之间的长距离依赖关系，替代了 RNN/LSTM 的循环递归和 CNN 的局部感受野。它解决的问题是：序列建模中的并行化缺失与长距离依赖衰减（梯度消失/遗忘）。机制上，Transformer 由多头注意力（Multi-Head Attention）、位置编码（Positional Encoding）和前馈网络（Feed-Forward Network）堆叠构成，每个子层后接残差连接与 LayerNorm。在整个计算机/AI 体系中，Transformer 是当前所有大语言模型（LLM）和大多数多模态模型的基石，也是生成式 AI 的主干架构。专业工程师必须掌握其底层原理，因为无论是 fine-tune、prompt engineering、模型部署推理还是架构选型，最终都要回归到对 token 如何被编码、注意力如何计算、KV Cache 如何起作用这些底层机制的理解上，否则对模型行为、内存占用和时延瓶颈的认知只能停留在接口调用层。

### 2. 底层原理剖析
Transformer 的输入是一个 token 序列，每个 token 先映射为一个 d 维向量（Embedding），然后加上位置编码得到初始隐状态 h。核心计算发生在多头自注意力（MHSA）：

1. 构造 Q/K/V：对每个 token 的隐状态 h，分别乘以三个可学习权重矩阵 W_Q、W_K、W_V，得到 query、key、value 向量。
2. 计算注意力分数：对每个 query 与所有 key 做点积，并除以 sqrt(d_k) 做缩放（防止 softmax 进入饱和区），得到原始注意力权重。
3. 归一化：对每行做 softmax，使权重和为 1。
4. 加权求和：将注意力权重作用于对应 value 向量并求和，得到该 token 的新表示。

多头则将 d 维分成 h 个头，每个头独立执行上述过程，最后拼接并经过输出投影 W_O。其本质是让不同头关注不同子空间的关系模式（如语法、语义、指代）。

位置编码（标准 Transformer 用三角函数，LLM 中常用 RoPE）为序列注入顺序信息，因为注意力本身是置换不变的，不显式区分 token 在序列中的先后次序。

FFN 是一个两层的 MLP（通常先升维再降维），对注意力输出做非线性变换，增强模型表征能力。每个子层（Attention 和 FFN）都采用残差连接：h = h + Sublayer(LayerNorm(h)) 或 Pre-Norm 变体（h = h + Sublayer(LayerNorm(h)) 在子层输入前做 Norm）。

与前端工程师熟悉的抽象对比：可以把 Transformer 当作一个极其强大的『状态更新器』，但它不像 React 那样有明确的单向数据流和生命周期。注意力机制更像是『动态组建的依赖图』，每个 token 的输出是所有其他 token 的加权结果，权重完全由数据驱动，没有像 DOM 那样的固定层级关系。若与 TypeScript 的类型系统类比：自注意力类似于『结构类型系统』——两个 token 的相关性不是预先声明的（名义类型），而是由它们的特征向量相似度（结构相似）动态推导的。同时，注意力矩阵的稀疏性与计算复杂度也类似前端状态管理中的 re-render 范围——默认全量计算，无法天然利用局部性。

### 3. 基础代码与实战验证
```text
以下是最简化的单头自注意力实现（纯 NumPy，不依赖任何深度学习框架），用于验证核心机制：

import numpy as np

def softmax(x):
    # 数值稳定 softmax：减去每行最大值防止 exp 溢出
    e = np.exp(x - np.max(x, axis=-1, keepdims=True))
    return e / np.sum(e, axis=-1, keepdims=True)

def self_attention(X, W_Q, W_K, W_V):
    """
    X: (seq_len, d_model) 输入序列，已包含位置编码
    W_Q, W_K, W_V: (d_model, d_k) 可学习投影矩阵，d_k 通常 = d_model / num_heads
    返回: (seq_len, d_k) 注意力输出
    """
    Q = X @ W_Q  # (seq_len, d_k) 每个 token 的 query
    K = X @ W_K  # (seq_len, d_k) 每个 token 的 key
    V = X @ W_V  # (seq_len, d_k) 每个 token 的 value
    
    d_k = K.shape[-1]
    scores = Q @ K.T / np.sqrt(d_k)  # (seq_len, seq_len) 所有 token 两两点积相似度，缩放防止梯度消失
    weights = softmax(scores)        # 每行归一化，得到注意力权重
    output = weights @ V             # (seq_len, d_k) 加权求和，每个 token 的新表示是全局信息的混合
    return output

# 验证：seq_len=3, d_model=4, d_k=4
np.random.seed(0)
X = np.random.randn(3, 4)
W_Q = np.random.randn(4, 4)
W_K = np.random.randn(4, 4)
W_V = np.random.randn(4, 4)

out = self_attention(X, W_Q, W_K, W_V)
print(out.shape)  # (3, 4)，输出长度不变，但每个 token 已聚合全局信息

# 关键验证点：第二行的输出等于所有 V 行按 attention weights 第 2 行加权求和
weights = softmax((X @ W_Q @ W_K.T @ X.T) / np.sqrt(4))
assert np.allclose(out, weights @ (X @ W_V))

# 可视化注意力权重
token_names = ['我', '喜欢', 'AI']
print('注意力权重矩阵：\n', np.round(weights, 2))
# 每行代表目标 token 对源 token 的关注强度，矩阵行和为 1

该代码直接展示了自注意力的本质：Q 决定“我要找什么”，K 决定“我是什么”，V 决定“我要贡献什么信息”。整个过程无循环、纯矩阵乘法，因此天然可并行，这是 Transformer 优于 RNN 的关键。
```

### 4. 常见误区与进阶思考
误区一：认为注意力机制能自动『看到』句子顺序。实际上，Self-Attention 本身是置换不变的，如果不加位置编码，模型看到的序列就是无序的集合。许多人将位置编码视为辅助信息，但它与 token embedding 同等关键。理解这一点后才能明白为什么 RoPE（旋转位置编码）的实现在长上下文泛化中如此重要。

误区二：认为 Transformer 的复杂度只在注意力层的 O(n²)。实际上，在 LLM 推理（生成）阶段，FFN（前馈网络）的参数量和计算量往往超过注意力层，而且主要瓶颈是显存带宽——每次生成一个 token 都要把所有参数过一遍。注意力层的 KV Cache 虽然显存占用随序列长度增长，但真正的开销分布在不同层间并不均衡。专业工程师在做服务调优时，必须实测而不是凭直觉。

思考题：在自回归生成（一个 token 一个 token 预测）的推理中，为什么 KV Cache 能显著加速？它是如何利用注意力计算的增量特性的？请用 Q/K/V 的矩阵维度动态变化来解释——具体而言，生成第 t 个 token 时，Q 是 1×d_k 的向量，但 K 和 V 是 t×d_k 的矩阵，缓存了此前所有 token 的 key 和 value。那么请你推导：如果没有 KV Cache，每次生成长度为 L 的序列，总计算量是多少？有 Cache 后又是多少？这能解释为什么现代 LLM 推理服务普遍使用 PagedAttention 管理 KV 缓存。
