---
title: "每日基础技术总结 · 2026-08-24 · Transformer 架构简述"
date: 2026-08-24 06:55:59
categories: [技术分享]
tags: ["技术分享", "AI 开发基础（LLM & Agent）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-24 · Transformer 架构简述

## 📚 今日主题

> **Transformer 架构简述**（AI 开发基础（LLM & Agent））

### 1. 核心概念速览
Transformer 是一种完全基于注意力机制（Attention Mechanism）的序列建模架构，由 Vaswani 等人在 2017 年《Attention Is All You Need》中提出。其本质是：将输入序列视为一组向量，通过可并行计算的查询-键-值（Query-Key-Value）三路线性投影与缩放点积注意力，直接建模序列中任意两个位置之间的依赖关系，从而替代 RNN/LSTM 中按时间步递推的隐状态传播机制。它解决的核心问题是：RNN 因时间步串行而无法并行训练，且长距离依赖会因梯度消失/爆炸而衰减。Transformer 的机制核心在于：依赖关系不是由网络结构（如循环边）隐式承载，而是由输入数据本身动态计算出的注意力权重显式承载。在整个计算机/AI 体系中，它是 GPT、BERT、T5 等大语言模型的底层骨架，位于深度学习架构层，向上承接预训练范式，向下衔接分布式训练与推理系统。专业工程师必须掌握它，因为 LLM 的上下文窗口、KV Cache、推理延迟、显存占用、Agent 工具调用等一切工程现象，最终都能追溯到注意力机制的计算复杂度与数据流；不理解它，就只能在 API 调用的表层做拼装，无法对模型行为做系统性推理与性能优化。

### 2. 底层原理剖析
底层运行机制按数据流拆解如下：
1. 输入表示：每个 token 经嵌入表映射为 d_model 维向量。由于注意力对位置不敏感（本质上是对集合的操作），必须显式加入位置编码（正弦波或可学习向量），使模型感知序列顺序。
2. 三路线性投影：输入 X（形状为 seq_len × d_model）分别乘以可学习矩阵 W_Q、W_K、W_V，得到 Q、K、V。注意：Q/K/V 不是输入本身，而是输入经线性变换后的激活值；真正可学习参数是 W_Q/W_K/W_V。
3. 缩放点积注意力：对每个 query，计算它与所有 key 的点积，得到相关性分数；除以 sqrt(d_k) 防止点积方差过大导致 softmax 进入饱和区；经 softmax 归一化为权重；再对 V 加权求和。公式：Attention(Q,K,V)=softmax(QK^T/√d_k)V。
4. 多头机制：将 d_model 切分为 h 个头，每个头独立做注意力，最后拼接并经 W_O 线性投影。本质是让模型在多个子空间中并行捕捉不同类型的依赖关系，类似卷积层中多个滤波器各司其职。
5. 前馈网络与残差/归一化：每个注意力子层后接一个两层 MLP（先升维再降维，激活函数为 ReLU/GELU）；每个子层都有残差连接（恒等映射保证梯度直通）和 LayerNorm（对每个 token 的特征维度做归一化，稳定训练）。
与前端已有概念的对比：RNN 处理序列的方式类似于 Array.prototype.reduce——每个时刻的隐状态必须等前一步计算完才能继续，串行且隐式携带历史信息；Transformer 处理序列的方式类似于 Promise.all 或全量事件广播——所有位置同时计算、彼此直接通信，历史信息不是逐步传递过来的，而是每个位置显式查询（query）所有位置（key）后聚合（value）得到的。这与『Java 接口（名义类型）vs TS 接口（结构类型）』的对比同构：两者表面都用于约束类型，但一个靠显式声明继承关系，一个靠结构兼容推断；RNN 与 Transformer 表面都用于建模序列，但一个靠隐式的时间步递推，一个靠显式的两两交互计算。

### 3. 基础代码与实战验证
```text
import numpy as np

def softmax(x, axis=-1):
    # 数值稳定 softmax：减去每行最大值，防止 exp 溢出
    e = np.exp(x - x.max(axis=axis, keepdims=True))
    return e / e.sum(axis=axis, keepdims=True)

def attention(Q, K, V):
    # Q, K, V 形状均为 (seq_len, d_k)
    d_k = K.shape[-1]
    # 1. 所有 token 两两求点积，得到 (seq_len, seq_len) 的相关性分数矩阵
    scores = Q @ K.T
    # 2. 除以 sqrt(d_k)：把点积结果的方差拉回 1 附近，
    #    防止 softmax 输入过大进入饱和区，导致梯度趋近于 0
    scores = scores / np.sqrt(d_k)
    # 3. 按行做 softmax，得到归一化的注意力权重（每行和为 1）
    weights = softmax(scores, axis=-1)
    # 4. 用权重对 V 加权求和：每个 token 的输出是全体 token 的 V 的凸组合
    return weights @ V, weights

def layer_norm(x, gamma, beta, eps=1e-6):
    # 对每个 token 的特征维度（最后一维）做归一化
    mean = x.mean(axis=-1, keepdims=True)
    var = x.var(axis=-1, keepdims=True)
    return (x - mean) / np.sqrt(var + eps) * gamma + beta

def encoder_block(x, W_q, W_k, W_v, W_o, W_1, b_1, W_2, b_2,
                  gamma_1, beta_1, gamma_2, beta_2):
    # x: (seq_len, d_model)，即已叠加位置编码的 token 向量序列
    # 子层 1：多头自注意力（此处以单头演示）
    Q = x @ W_q   # 当前 token 作为查询方：向所有位置询问『你与我的相关性』
    K = x @ W_k   # 所有 token 作为键：供查询方做相似度匹配
    V = x @ W_v   # 所有 token 作为值：真正被聚合的信息载体
    attn_out, _ = attention(Q, K, V)
    attn_out = attn_out @ W_o   # 输出投影：将注意力结果重新映射到 d_model 空间
    x = x + attn_out            # 残差连接：让梯度恒等直通，避免深层网络退化
    x = layer_norm(x, gamma_1, beta_1)  # 对每个位置的特征做归一化，稳定训练
    # 子层 2：逐位置前馈网络（对每个 token 独立做两次线性变换 + ReLU）
    ff_out = np.maximum(x @ W_1 + b_1, 0) @ W_2 + b_2
    x = x + ff_out              # 第二个残差连接
    x = layer_norm(x, gamma_2, beta_2)
    return x

# 验证：d_model=8，序列长度 4
np.random.seed(0)
x = np.random.randn(4, 8)
W_q = np.random.randn(8, 8) * 0.1
W_k = np.random.randn(8, 8) * 0.1
W_v = np.random.randn(8, 8) * 0.1
W_o = np.random.randn(8, 8) * 0.1
W_1 = np.random.randn(8, 32) * 0.1
b_1 = np.zeros(32)
W_2 = np.random.randn(32, 8) * 0.1
b_2 = np.zeros(8)
gamma_1 = np.ones(8); beta_1 = np.zeros(8)
gamma_2 = np.ones(8); beta_2 = np.zeros(8)
out = encoder_block(x, W_q, W_k, W_v, W_o, W_1, b_1, W_2, b_2,
                    gamma_1, beta_1, gamma_2, beta_2)
print(out.shape)  # (4, 8)：每个位置的输出都聚合了全序列信息
```

### 4. 常见误区与进阶思考
误区一：把『注意力权重』当作可学习参数。实际上 W_Q/W_K/W_V 这些投影矩阵才是可学习参数，注意力权重是对当前输入动态计算出的激活值——同一个模型面对不同输入会产生不同注意力分布。这类似于前端中『组件的 props 是外部传入的数据，而非组件内部定义的常量』；把动态计算值误认为静态参数，会从根本上误判模型的行为与可解释性。
误区二：认为 Transformer 的并行性等于低计算量。自注意力的时间/空间复杂度是 O(n²)（n 为序列长度），并行指的是『时间步维度』上的并行，而非计算总量变少。LLM 推理时的 KV Cache 正是为了缓存历史 K/V 以避免每个 token 重复计算，但预填充阶段仍要对全量序列做 O(n²) 的注意力计算。忽略这一点，就无法理解长上下文为何昂贵、滑动窗口与稀疏注意力为何存在。
思考题：在因果语言模型（decoder-only）中，QK^T 之后、softmax 之前必须施加一个上三角掩码，把未来位置的分数置为 -inf。请从『信息流方向』与『训练-推理一致性』两个角度分析：如果移除该掩码，训练时会发生什么？推理时的自回归生成是否还成立？理解这一点，才说明你真正读懂了注意力矩阵每一行的语义——第 i 行的权重分布决定了第 i 个 token 能看到哪些历史信息。
