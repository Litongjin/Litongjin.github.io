---
title: "每日基础技术总结 · 2026-09-10 · Transformer 架构简述"
date: 2026-09-10 07:01:19
categories: [技术分享]
tags: ["技术分享", "AI 开发基础（LLM & Agent）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-10 · Transformer 架构简述

## 📚 今日主题

> **Transformer 架构简述**（AI 开发基础（LLM & Agent））

### 1. 核心概念速览
Transformer 是一种基于纯注意力机制（Attention Mechanism）的序列到序列（Seq2Seq）架构，由 Vaswani 等人在 2017 年提出。其本质是用『全局依赖建模』替代循环神经网络（RNN）的『逐步递推状态传递』，通过并行计算所有位置间的加权交互来捕获上下文关系。它解决的问题是：长距离依赖中的梯度消失/爆炸、顺序计算的不可并行性，以及传统注意力机制无法直接作为独立层堆叠的问题。在整个计算机/AI 体系中，Transformer 是当前大语言模型（LLM）、多模态模型（如 ViT）的骨干网络，也是 Agent 系统中感知、规划、推理模块的核心算子。专业工程师必须掌握它，因为现代 AI 应用的性能优化、服务部署、推理加速、Token 成本控制，其底层都直接受 Transformer 的注意力复杂度、KV Cache、上下文窗口等机制约束，理解架构才能从原理上理解显存占用、延迟瓶颈和模型行为。

### 2. 底层原理剖析
Transformer 的核心是可缩放的点积注意力（Scaled Dot-Product Attention）。给定查询矩阵 Q、键矩阵 K、值矩阵 V，注意力输出为 Attention(Q,K,V) = softmax(QK^T / sqrt(d_k)) V。其中 d_k 是键的维度，除以 sqrt(d_k) 是为了防止点积值过大导致 softmax 梯度消失。Q、K、V 均由输入序列 X 通过三个可学习的线性投影得到：Q = XW_Q, K = XW_K, V = XW_V。多头注意力（Multi-Head Attention）将 Q、K、V 切分为 h 个维度为 d_k/h 的子空间，分别计算注意力后拼接，再经线性投影，使得模型能在不同表示子空间捕获不同类型的依赖关系。Transformer 的每个编码器/解码器层包含两个子层（解码器为三个）：多头注意力、前馈网络（FFN，即两层线性变换+ReLU 或 GELU），每个子层后添加残差连接（Residual Connection）和层归一化（Layer Normalization）。位置编码（Positional Encoding）是必须的，因为注意力是位置不变的（Permutation Invariant），不引入位置信息则序列顺序完全丢失；原始实现使用正弦/余弦函数生成固定位置编码，后续研究使用可学习位置编码或旋转位置编码（RoPE）。与前端工程师熟悉的『接口』概念对比：Java 的接口是编译期约束类型行为的契约，TypeScript 的接口是结构类型系统的静态约定，而 Transformer 的『接口』实质是张量的形状约定——输入 [batch, seq_len, d_model]，经各层后输出相同形状，这种约定的执行发生在运行时，且由矩阵乘法的维度匹配强制，而非编译期类型检查。Transformer 的核心计算流程如下：输入 Token Embedding [B, T, d] → 加位置编码 → 重复 L 次：注意力（QKV 投影、点积、缩放、Softmax、加权求和、输出投影）→ 残差 → LayerNorm → FFN → 残差 → LayerNorm → 输出。自回归解码时需使用因果掩码（Causal Mask）保证位置 i 只能 attend 到 j ≤ i。

### 3. 基础代码与实战验证
```text
以下为极简纯 Python + NumPy 实现单层单头 Transformer 编码器的核心部分，不依赖框架，用于验证注意力机制与残差归一化流程。

import numpy as np

def softmax(x):
    # 沿最后一个维度做稳定 softmax，减去最大值防止 exp 溢出
    e = np.exp(x - np.max(x, axis=-1, keepdims=True))
    return e / np.sum(e, axis=-1, keepdims=True)

def layer_norm(x, eps=1e-5):
    # 对特征维度做标准化：均值为0，方差为1，再乘 gamma 加 beta（此处简化为恒等）
    mean = np.mean(x, axis=-1, keepdims=True)
    var = np.var(x, axis=-1, keepdims=True)
    return (x - mean) / np.sqrt(var + eps)

def attention(Q, K, V):
    # Q, K, V shape: [seq_len, d_k] 或 [batch, seq_len, d_k]
    d_k = Q.shape[-1]
    # 点积：Q 与 K 的转置相乘，得到 [seq_len, seq_len] 的注意力分数矩阵
    scores = np.matmul(Q, K.transpose(0, 2, 1) if K.ndim == 3 else K.T) / np.sqrt(d_k)
    # 对最后一个维度（键的位置）做 softmax，得到权重
    weights = softmax(scores)
    # 加权求和：weights 与 V 相乘，得到新的表示
    return np.matmul(weights, V)

class TransformerLayer:
    def __init__(self, d_model, d_ff):
        self.d_model = d_model
        # 线性投影矩阵，此处用随机初始化，实际训练需要学习
        self.W_Q = np.random.randn(d_model, d_model) * 0.01
        self.W_K = np.random.randn(d_model, d_model) * 0.01
        self.W_V = np.random.randn(d_model, d_model) * 0.01
        self.W_O = np.random.randn(d_model, d_model) * 0.01
        self.W_1 = np.random.randn(d_model, d_ff) * 0.01
        self.W_2 = np.random.randn(d_ff, d_model) * 0.01

    def forward(self, x):
        # x shape: [batch, seq_len, d_model]
        # 1. 自注意力子层
        Q = np.matmul(x, self.W_Q)
        K = np.matmul(x, self.W_K)
        V = np.matmul(x, self.W_V)
        attn_out = attention(Q, K, V)
        attn_out = np.matmul(attn_out, self.W_O)  # 输出投影
        # 2. 残差连接 + 层归一化（Post-Norm 风格）
        x = layer_norm(x + attn_out)
        # 3. 前馈网络子层
        ff_out = np.matmul(x, self.W_1)
        ff_out = np.maximum(ff_out, 0)  # ReLU 激活
        ff_out = np.matmul(ff_out, self.W_2)
        # 4. 残差连接 + 层归一化
        x = layer_norm(x + ff_out)
        return x

# 验证：输入 2 个样本，每个 3 个 token，特征维度 4
np.random.seed(0)
x = np.random.randn(2, 3, 4)
layer = TransformerLayer(d_model=4, d_ff=8)
out = layer.forward(x)
print(out.shape)  # 输出 (2, 3, 4)，形状不变，验证残差连接保证了维度稳定

关键说明：
- 注意力分数的缩放因子 1/sqrt(d_k) 是保证训练稳定的核心，若 d_k 增大而不缩放，softmax 输入区域进入饱和区，梯度极小。
- 层归一化在残差之后（Post-LN）是原始架构方式；现代实现多用 Pre-LN 以提升训练稳定性。
- 未实现多头、位置编码、因果掩码，但上述单头逻辑是基础，多头即对多个子空间并行执行此过程后拼接。
```

### 4. 常见误区与进阶思考
常见误区一：认为 Attention 能无条件捕捉任意长距离依赖。实际上，自注意力是全局的，但它的有效性受限于训练分布中的模式，且 Softmax 后权重往往集中于少数位置，过长的上下文会导致注意力分散或集中于局部；同时，QK^T 的计算复杂度为 O(n^2)（n 为序列长度），这在实际工程中导致长上下文必须依赖稀疏注意力、窗口注意力或 KV Cache 等技巧，并非简单堆层能解决。常见误区二：混淆『位置编码』的作用与『 Token Embedding』。位置编码必须与 Token Embedding 融合（加或拼接）后才进入注意力，但注意力本身并不感知顺序，位置编码只是打破置换不变性；若去掉位置编码，模型无法区分『我爱你』和『你爱我』，甚至等价于 Bag-of-Words 加全局池化。深度思考题：在自回归解码阶段，为什么需要因果掩码？如果去掉因果掩码，训练时每个位置都能看到未来 Token，模型可以轻易预测而无需理解因果关系，那么模型在推理时（只能看到过去）将严重失效；但请进一步解释：因果掩码具体在哪一步作用于注意力分数矩阵？它影响的是 softmax 的输入还是输出？请在数学层面上推导 mask 为 -inf 时，softmax 的输出如何保证位置 i 对 j>i 的权重严格为 0，以及为什么不能直接将对应位置的 QK^T 置为 0（因为 softmax 分母仍会包含该位置，导致权重不为零）。这个问题的答案决定了你能否真正理解现代 LLM 推理引擎中的 KV Cache 优化边界。
