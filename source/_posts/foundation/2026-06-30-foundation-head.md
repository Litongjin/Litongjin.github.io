---
title: "每日基础技术总结 · 2026-06-30 · 多头注意力中 head 维度与模型维度的关系及切分方式"
date: 2026-06-30 08:00:00
categories: [技术分享]
tags: ["技术分享", "AI 开发基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-06-30 · 多头注意力中 head 维度与模型维度的关系及切分方式

## 📚 今日主题

> **多头注意力中 head 维度与模型维度的关系及切分方式**（AI 开发基础）

### 1. 核心概念速览
多头注意力（Multi-Head Attention）中，模型维度（d_model，即输入/输出的特征维度）与每个头的维度（head_dim，通常记为 d_k 或 d_v）之间存在严格的整数倍关系：d_model = num_heads × head_dim。本质是将一个高维向量空间切分为多个低维子空间，每个头独立地在子空间内计算注意力，最后拼接并线性投影回 d_model。切分方式是对 Q、K、V 的最后一维（特征维）进行连续分块（chunk），而非间隔采样；每个头对应一个连续的子向量区间。它解决的是单一注意力分布只能建模一种关系模式的问题，多头机制让模型在不同子空间中并行捕捉多种依赖模式。该机制位于 Transformer 架构的核心位置，是自监督预训练、跨模态对齐、长序列建模等一切现代 AI 系统的基石。专业工程师必须掌握它，因为所有部署优化（KV Cache、张量并行、FlashAttention）都建立在对维度切分和重组的精确理解之上，任何维度错位或头数配置错误都会导致模型输出崩溃。

### 2. 底层原理剖析
底层运行机制如下：给定输入 X（形状 [batch, seq_len, d_model]），通过三个权重矩阵 W_Q、W_K、W_V（形状均为 [d_model, d_model]）线性变换得到 Q、K、V，形状保持 [batch, seq_len, d_model]。随后进行切分：将 d_model 按 head_dim 大小切成 num_heads 份，每一份对应一个头。以 PyTorch 的 view 操作为例，先将 Q 从 [B, L, D] reshape 为 [B, L, H, D/H]，然后转置为 [B, H, L, D/H]。这里的 H = num_heads，D/H = head_dim。每个头独立计算 attention(Q_h, K_h, V_h) = softmax(Q_h K_h^T / sqrt(head_dim)) V_h，得到 [B, H, L, D/H] 的输出。最后将 H 个头的输出拼接：先转置回 [B, L, H, D/H]，再 reshape 为 [B, L, D]，通过输出投影矩阵 W_O（[d_model, d_model]）得到最终结果。关键点：切分是沿特征维度的均匀划分，每个头只看到原始特征的局部片段，但所有头共享同一组输入。这与前端工程师熟悉的 TypeScript 接口和 Java 接口的对比有异曲同工：TS 接口是结构化类型（structural typing），关注形状是否匹配，但不会限制实现方式；Java 接口是名义类型（nominal typing），必须显式声明 implements。多头注意力中的维度切分类似结构化类型——只要总维度能被头数整除，切分方式就是确定的，运行时不会检查头的‘语义’；而权重矩阵的初始化与训练过程则像名义类型——每个头通过不同初始化学到不同语义，头之间没有显式交互，完全靠数据驱动分化。更本质地说，切分是静态的、确定性的张量操作，而头的语义分化是动态的、涌现的。这种‘静态形状约束 + 动态语义分化’的组合，与前端中‘接口定义形状 + 实现类决定行为’的模型完全对应。

### 3. 基础代码与实战验证
```text
以下为极简 PyTorch 代码，验证切分与拼接过程（不依赖 Transformer 封装）：

import torch
import torch.nn.functional as F

B, L, D = 2, 4, 8   # batch=2, seq_len=4, d_model=8
H = 2               # 头数=2，则 head_dim = D // H = 4

# 模拟 QKV（已通过线性层得到）
Q = torch.randn(B, L, D)
K = torch.randn(B, L, D)
V = torch.randn(B, L, D)

# 切分：将最后维度 D 切成 H 份，每份连续 head_dim 个元素
# 先 view 成 [B, L, H, head_dim]，再转置为 [B, H, L, head_dim]
Q = Q.view(B, L, H, D // H).transpose(1, 2)  # 形状 [B, H, L, head_dim]
K = K.view(B, L, H, D // H).transpose(1, 2)
V = V.view(B, L, H, D // H).transpose(1, 2)

# 每个头独立计算缩放点积注意力
scores = torch.matmul(Q, K.transpose(-2, -1)) / (D // H) ** 0.5  # [B, H, L, L]
attn = F.softmax(scores, dim=-1)
context = torch.matmul(attn, V)  # [B, H, L, head_dim]

# 拼接：先转置回 [B, L, H, head_dim]，再 view 回 [B, L, D]
context = context.transpose(1, 2).contiguous().view(B, L, D)
# 输出投影（可省略，但体现完整机制）
W_O = torch.randn(D, D)
output = torch.matmul(context, W_O)  # [B, L, D]

注释：view 不改变内存布局，只是重新解释形状；transpose 交换维度后需 contiguous() 才能 view，因为转置后内存非连续。切分时每个头获取的是原始向量的连续区间（0-3 和 4-7），这保证了计算的可并行性和缓存友好性。
```

### 4. 常见误区与进阶思考
常见误区 1：认为每个头有独立的权重矩阵且输出维度是 d_model。实际上所有头共享同一份 QKV 输入，但通过同一个权重矩阵得到 Q/K/V 后切分；每个头不是独立的线性层，而是同一个线性层输出的切片。若误解为每个头单独做线性变换，会导致参数计算错误（实际参数为 4×d_model²，而非 3×d_model²+3×H×head_dim²）。常见误区 2：混淆 num_heads 与 head_dim 的约束关系，认为 head_dim 可以任意设置。实际必须满足 d_model % num_heads == 0，否则无法均分；一些框架允许不同头有不同维度（如 GQA/MQA），但那是变体，标准多头注意力中所有 head_dim 相等。

进阶思考题：在 KV Cache 推理优化中，我们通常将 K 和 V 的缓存按头维度存储为 [B, H, L, head_dim]。如果模型采用张量并行（Tensor Parallelism）将头切分到多个 GPU 上，每个 GPU 只负责一部分头。此时如果某个注意力头对应的输入位置（seq_len 维度）被另一个 GPU 计算，那么该 GPU 的 KV Cache 应该存储哪些张量？请从切分方式的角度说明：张量并行下每个 GPU 的 QKV 输入是如何切分的，与标准多头切分有何异同？这决定了通信模式是 All-to-All 还是 AllReduce。
