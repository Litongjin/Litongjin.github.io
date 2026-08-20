---
title: "每日基础技术总结 · 2026-06-30 · Transformer 位置编码：正弦编码与旋转位置编码 RoPE"
date: 2026-06-30 08:00:00
categories: [技术分享]
tags: ["技术分享", "AI 开发基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-06-30 · Transformer 位置编码：正弦编码与旋转位置编码 RoPE

## 📚 今日主题

> **Transformer 位置编码：正弦编码与旋转位置编码 RoPE**（AI 开发基础）

### 1. 核心概念速览
位置编码是Transformer中打破自注意力排列等变性的输入增强机制。自注意力对序列顺序不敏感，任意置换输入会得到对应置换的输出，因此必须显式注入顺序信息。正弦编码（Sinusoidal）使用不同频率的正弦/余弦函数生成绝对位置向量，与token嵌入相加，提供绝对位置先验。旋转位置编码（RoPE）通过将query和key向量按维度分组后施加与绝对位置相关的旋转变换，使注意力分数中的内积项自然成为相对位置差的函数，提供相对位置先验。它在Transformer输入表示层中占据核心位置。专业工程师必须掌握，因为RoPE是现代长上下文模型（如LLaMA、Mistral）的基础设施，直接影响外推能力、上下文窗口扩展和训练动态。

### 2. 底层原理剖析
正弦编码：位置pos、维度i，频率w_i=1/10000^(2i/d)。定义PE(pos,2i)=sin(pos*w_i)，PE(pos,2i+1)=cos(pos*w_i)。关键性质：PE(pos+k)可由PE(pos)经线性变换得到（三角恒等式），因此理论上包含相对位置信息。但它以加法方式融入embedding，且是绝对位置，多层注意力中相对信息需间接提取。
RoPE：将d维向量拆成d/2个二维平面。对第j个平面，旋转角度θ=pos*base^(-2j/d)。设q和k的绝对位置为p_q和p_k，则内积 <R(p_q)q, R(p_k)k> = q^T R(p_k-p_q) k，只依赖相对位置。实现时无需构造d×d矩阵，对每个二维平面应用旋转矩阵 [cosθ, -sinθ; sinθ, cosθ] 即可。本质是乘性编码，不改变向量模长，只改变方向。
对比前端概念：正弦编码类似在DOM节点上显式设置data-position属性；RoPE则类似通过事件流中target与currentTarget的深度差计算相对层级，或者类似CSS的transform: rotate，不修改文档流位置，但改变局部坐标系方向，从而影响与其他元素的相对关系。核心差异：正弦编码是加性的绝对位置先验，RoPE是乘性的相对位置先验。

### 3. 基础代码与实战验证
```text
以下使用NumPy实现正弦编码和RoPE，并验证RoPE的内积只依赖相对位置（d=4）。

import numpy as np

def sinusoidal_encoding(max_pos, d_model):
    pe = np.zeros((max_pos, d_model))
    for pos in range(max_pos):
        for i in range(d_model // 2):
            theta = pos / (10000 ** (2 * i / d_model))
            pe[pos, 2 * i] = np.sin(theta)
            pe[pos, 2 * i + 1] = np.cos(theta)
    return pe

def rope(x, pos, base=10000):
    dim = x.shape[0]
    freqs = 1.0 / (base ** (np.arange(0, dim, 2) / dim))
    angles = pos * freqs
    x_pair = x.reshape(-1, 2)
    cos = np.cos(angles)
    sin = np.sin(angles)
    rotated = np.stack(
        [x_pair[:, 0] * cos - x_pair[:, 1] * sin,
         x_pair[:, 0] * sin + x_pair[:, 1] * cos],
        axis=-1
    )
    return rotated.reshape(-1)

rng = np.random.default_rng(0)
q = rng.normal(size=4)
k = rng.normal(size=4)
p1, p2 = 3, 7
inner_abs = np.dot(rope(q, p1), rope(k, p2))
inner_rel = np.dot(rope(q, 0), rope(k, p2 - p1))
print(inner_abs, inner_rel)

注释：
- sinusoidal_encoding 中，频率随维度指数衰减，偶维用正弦、奇维用余弦，每个位置形成独立向量。
- rope 中，将向量重组成 [dim/2, 2] 的二维向量，按位置角度旋转；内积计算时，两个绝对旋转的内积等价于一个不旋转与一个旋转角度差的内积，从而验证相对位置。
```

### 4. 常见误区与进阶思考
误区1：将RoPE归类为绝对位置编码。RoPE的旋转角度确实依赖绝对位置，但注意力内积经过旋转变换后只保留相对位置差，因此其本质是相对位置编码。正弦编码才是绝对位置编码，虽然它通过线性变换也能表达相对信息。
误区2：认为RoPE天然支持任意长度外推。RoPE的角度与位置成正比，当位置超出训练范围时，旋转角可能进入模型未见过的区间，导致注意力分数退化。实际工程中需要配合NTK-aware缩放、YaRN或ALiBi等策略。
思考题：设RoPE的旋转角度为θ_p = p^2 * w_i（平方位置），推导此时注意力内积的表达式，并说明为什么它无法表示相对位置。这个例子揭示了线性相位对于相对位置编码的本质性作用。
