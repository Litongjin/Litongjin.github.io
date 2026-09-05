---
title: "每日基础技术总结 · 2026-09-06 · Temperature / Top-P 等参数对输出的影响"
date: 2026-09-06 07:01:43
categories: [技术分享]
tags: ["技术分享", "AI 开发基础（LLM & Agent）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-06 · Temperature / Top-P 等参数对输出的影响

## 📚 今日主题

> **Temperature / Top-P 等参数对输出的影响**（AI 开发基础（LLM & Agent））

### 1. 核心概念速览
Temperature 与 Top-P 是控制自回归语言模型解码策略的核心采样参数，本质是调整下一个 token 概率分布的‘锐度’与‘截断范围’。Temperature 通过对 logits 除以温度系数 T 后施加 softmax，改变分布熵：T 越小（接近 0），分布越尖锐，近似贪心解码；T 越大，分布越平坦，低概率 token 被赋予更高采样机会。Top-P（nucleus sampling）则基于累积概率阈值 p，在每一步动态截断候选集：将概率按降序累加至 p，仅从该最小集合中采样，从而丢弃长尾低概率 token。二者解决的核心问题是：在‘确定性/连贯性’与‘多样性/创造性’之间做可控权衡，是平衡生成质量与重复率的底层旋钮。该知识点位于 LLM 推理与生成阶段，属于解码策略层，独立于模型权重训练；专业工程师必须掌握，因为它是任何生成式 AI 产品调优、成本控制、可复现性设计的基础，直接决定输出的业务可用性与风险边界。

### 2. 底层原理剖析
自回归生成模型在每一步输出词汇表 V 上的 logits z = (z1, z2, ..., z|V|)。原始 softmax 定义为 P(wi) = exp(zi) / Σj exp(zj)。Temperature 的机制是在 softmax 前对 logits 缩放：P_T(wi) = exp(zi / T) / Σj exp(zj / T)。当 T→0+，exp((zi - zmax)/T) 中最大 logits 对应的概率趋近 1，其余趋近 0，等价于 argmax；当 T→∞，所有 logits 差异被抹平，分布趋近均匀。由于 logits 可能为负且分数量级差异大，实际实现中通常先减去最大值 zmax 做数值稳定，再除以 T。Top-P 的机制是在原始（或经 Temperature 变换后的）概率分布上，将 token 按概率降序排列，依次累加概率直至累加和 ≥ p，该前缀集合即为候选集，然后在候选集内重新归一化并采样。本质是过滤掉累积概率低于 p 的长尾 token，避免低概率噪声被采样。与 Temperature 的区别：Temperature 是连续地改变所有 token 的相对概率，不改变 rank 顺序（仅改变相对差距）；Top-P 是离散地截断候选集，可能完全排除某些低概率 token。二者可组合使用，常见顺序为先应用 Temperature 再应用 Top-P。与前端知识的对比：可将 Temperature 类比为 CSS 的 opacity——对整个图层（概率分布）做全局均匀缩放变换；Top-P 类比为 Array.filter()——按条件（累积概率阈值）删除不满足条件的元素，再对剩余元素做归一化（reduce + map）。更本质地，Temperature 是概率空间上的可逆热力学变换（熵调节），Top-P 是支持域（support）上的集合投影——前者改变分布的‘平滑度’，后者改变分布的‘支集大小’。二者共同将模型的原始信号（logits）映射为用户可感知的生成行为（确定性/多样性）。

### 3. 基础代码与实战验证
```text
以下为纯 Python 伪代码/最小实现，不依赖任何深度学习框架，精确演示 Temperature 与 Top-P 作用于一个 5 维 logits 向量的底层过程。

import math
import random

logits = [3.0, 1.0, 0.5, 0.1, -0.5]  # 原始模型输出（未归一化的分数）

# 1. Temperature 缩放
def temperature_scale(logits, T):
    # T 必须大于 0；T 越小分布越尖锐，T 越大分布越平坦
    scaled = [z / T for z in logits]  # 逐元素除以温度系数
    # 数值稳定：减去最大值防止 exp 溢出，不影响 softmax 结果
    m = max(scaled)
    exp_s = [math.exp(z - m) for z in scaled]
    total = sum(exp_s)
    probs = [e / total for e in exp_s]  # 归一化为合法概率分布
    return probs

# 2. Top-P 截断（nucleus sampling）
def top_p_filter(probs, p):
    # 将 token 下标按概率降序排序
    indexed = sorted(enumerate(probs), key=lambda x: x[1], reverse=True)
    cumulative = 0.0
    keep = []
    for idx, prob in indexed:
        cumulative += prob
        keep.append(idx)
        if cumulative >= p:  # 累积概率超过阈值 p 即停止扩展候选集
            break
    # 在候选集内重新归一化（保留原始概率值，再除以候选集概率总和）
    selected_probs = [probs[i] for i in keep]
    s = sum(selected_probs)
    normalized = {i: probs[i] / s for i in keep}  # 重新归一化后的采样分布
    return normalized

# 验证：T=1 时为原始 softmax 分布
probs_T1 = temperature_scale(logits, 1.0)
print('T=1:', probs_T1)

# 验证：T=0.5 时，高概率 token 概率更大（分布更尖锐）
probs_T05 = temperature_scale(logits, 0.5)
print('T=0.5:', probs_T05)

# 验证：T=2.0 时，分布更平坦
probs_T2 = temperature_scale(logits, 2.0)
print('T=2:', probs_T2)

# Top-P: 取 T=1 的分布，p=0.8，则只保留累积概率到 0.8 的前几个 token，丢弃长尾
filtered = top_p_filter(probs_T1, 0.8)
print('Top-P=0.8 候选分布:', filtered)

# 实际采样：从过滤后的分布中随机抽取一个下标
# 注意：真实框架中会使用可复现的随机数生成器（如 torch.multinomial / random.choices）
```

### 4. 常见误区与进阶思考
误区 1：认为 Temperature 越低越‘聪明’或越‘正确’。实际上 Temperature 仅控制采样的随机性，不改变模型内部的知识和推理能力。T 接近 0 时输出更可能选择 logits 最高的 token，但该 token 在语义上未必是最优的——尤其是开放域生成时，过低的 T 会导致重复、僵化、陷入局部循环；过高的 T 会产生逻辑断裂和幻觉。正确做法是根据任务性质调节：代码/数学等确定性任务用低 T（0~0.3），创意写作/头脑风暴用高 T（0.7~1.0），且必须配合其他约束（repetition_penalty、frequency_penalty）联合调优。误区 2：将 Top-P 与 Top-K 混为一谈，或认为 Top-P 与 Temperature 互斥。Top-K 是固定候选集大小，Top-P 是动态截断，后者在不同概率分布形状下自适应候选规模，更健壮。二者与 Temperature 是正交的变换维度：Temperature 改变分布形状，Top-P/Top-K 裁剪候选集，实际推理时三者可同时使用，且顺序有讲究（先缩放再截断）。若只调 Top-P 而不动 Temperature，生成结果的多样性变化是离散的，容易出现跳变；组合使用时需要理解每个参数的作用域，避免叠加后过度随机或过度确定。

深度思考题：给定两个 token 的 logits 分别为 [10, 9] 和 [1, 0]，二者概率差在原始 softmax 下完全相同（exp(1) 倍数）。若分别施加 T=0.1，请问两个场景的采样概率差异是否相同？若施加 T=2 呢？画出二者概率比随 T 变化的函数，并解释粗糙的 logits 尺度（logits 绝对值大小）与 Temperature 之间的交互如何影响实际生成行为——这是理解为何不能跨模型直接复用同一组 temperature 参数的关键。
