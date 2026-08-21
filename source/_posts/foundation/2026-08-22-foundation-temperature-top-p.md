---
title: "每日基础技术总结 · 2026-08-22 · Temperature / Top-P 等参数对输出的影响"
date: 2026-08-22 07:00:49
categories: [技术分享]
tags: ["技术分享", "AI 开发基础（LLM & Agent）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-22 · Temperature / Top-P 等参数对输出的影响

## 📚 今日主题

> **Temperature / Top-P 等参数对输出的影响**（AI 开发基础（LLM & Agent））

### 1. 核心概念速览
Temperature 和 Top-P 是 LLM 解码阶段的采样超参数，直接作用于模型输出的概率分布。Temperature 通过缩放 logits（未归一化得分）调整 softmax 的锐度：temperature > 1 使分布趋于均匀，增加随机性；temperature < 1 使分布集中于高概率 token，减少随机性；temperature=0 时退化为贪心解码（argmax）。Top-P（核采样）则在 softmax 后选择累积概率刚好超过 P 的最小 token 子集，并将该子集的概率重新归一化，从而动态截断低概率尾部。它们解决的问题是平衡生成文本的多样性与连贯性，避免采样到不合理的 token。机制发生在概率采样阶段，而非模型前向推理阶段，属于解码策略的一部分。专业工程师必须掌握其数学本质，才能在实际应用中合理调节参数，避免盲目调参导致输出质量劣化。

### 2. 底层原理剖析
底层机制分为两步：
1. Temperature 缩放：给定 logits z，缩放后的 logits 为 z_i / T，其中 T 为 temperature。然后通过 softmax 得到概率 p_i = exp(z_i/T) / Σ_j exp(z_j/T)。T 越小，logits 的相对差异被放大，softmax 分布越尖锐；T 越大，差异被压缩，分布越平坦。
2. Top-P 核采样：将概率降序排序，计算累积概率，选择最小的集合 S 使得 Σ_{i∈S} p_i ≥ P，然后对 S 中的概率重新归一化，最后从该分布中采样。
伪代码：
logits = model(x)
if T != 1: logits = logits / T
probs = softmax(logits)
if P < 1:
    sorted_probs, indices = sort(probs, descending)
    cumsum = cumulative_sum(sorted_probs)
    keep = indices[cumsum - sorted_probs < P]  # 保留累积概率小于 P 的，并加上第一个超过的
    # 或：找到第一个 cumsum >= P 的位置，保留该位置及之前的 token
    probs = renormalize(probs[keep])
token = sample(probs)
与前端工程师熟悉的概念对比：Temperature 对应数值数组的整体缩放操作（如将数组中每个元素乘以一个系数），Top-P 对应条件过滤操作（如保留满足条件的元素后重新计算占比）。两者的本质区别在于：Temperature 在 softmax 之前调整原始得分，影响所有 token 的相对概率；Top-P 在 softmax 之后截断概率分布的尾部，只影响低概率 token 是否进入候选集。它们不是等价的，组合使用时先缩放后截断，顺序敏感。

### 3. 基础代码与实战验证
```text
以下代码用 NumPy 实现 Temperature 和 Top-P 的完整采样过程，不依赖任何深度学习框架：

import numpy as np

def sample_with_temperature_top_p(logits, temperature=1.0, top_p=1.0):
    # logits: 模型输出的原始得分向量，形状为 (vocab_size,)
    logits = np.asarray(logits, dtype=np.float64)

    # Step 1: Temperature 缩放
    if temperature != 1.0:
        logits = logits / temperature  # 缩放 logits，温度越低，logits 间差距越大

    # Step 2: Softmax 转换为概率分布
    exp_logits = np.exp(logits - np.max(logits))  # 减去最大值防止指数溢出
    probs = exp_logits / np.sum(exp_logits)

    # Step 3: Top-P 核采样
    if top_p < 1.0:
        sorted_indices = np.argsort(probs)[::-1]  # 概率降序索引
        sorted_probs = probs[sorted_indices]
        cumsum_probs = np.cumsum(sorted_probs)  # 累积概率

        # 找到第一个累积概率 >= top_p 的位置，保留该位置及之前的 token
        # np.searchsorted(cumsum_probs, top_p, side='left') 返回第一个 >= top_p 的索引
        keep_count = np.searchsorted(cumsum_probs, top_p, side='left') + 1  # 至少保留一个 token
        keep_indices = sorted_indices[:keep_count]

        # 重新归一化，只保留候选 token 的概率
        new_probs = np.zeros_like(probs)
        new_probs[keep_indices] = probs[keep_indices]
        probs = new_probs / np.sum(new_probs)

    # Step 4: 从最终概率分布中采样一个 token 索引
    return np.random.choice(len(probs), p=probs)

验证方式：准备一组 logits，如 [1.0, 2.0, 3.0, 4.0]，分别设置 temperature=0.1、1.0、10.0，观察采样概率变化；设置 top_p=0.8 观察低概率 token 被过滤。实际运行时，temperature 越小，高概率 token 被采样频率越高；top_p 越小，候选集越小。
```

### 4. 常见误区与进阶思考
常见误区1：认为 temperature 可以控制模型的知识或确定性。实际上 temperature 只影响采样的概率分布形状，模型参数和推理结果不变；即使 temperature=0 也只是采用最大概率 token，不代表该 token 一定正确。
常见误区2：认为 top_p 与 temperature 效果等价，可以互相替代。实际上两者作用在不同阶段：temperature 在 softmax 前对 logits 整体缩放，影响所有 token 的分布陡峭度；top_p 在 softmax 后动态截断低概率 token，只改变候选集。两者组合时，temperature 会先改变概率分布，再被 top_p 截断，因此对最终候选集大小有非线性影响。
思考题：给定一组固定的 logits 和固定的 top_p 值（如 0.9），当 temperature 从 1.0 逐渐增大到 10.0 时，最终被保留的 token 数量会如何变化？请从数学上解释为什么。这能验证你是否真正理解温度对分布形状的影响以及 top_p 截断的动态性。
