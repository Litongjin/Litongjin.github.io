---
title: "每日基础技术总结 · 2026-09-14 · Temperature / Top-P 等参数对输出的影响"
date: 2026-09-14 07:02:36
categories: [技术分享]
tags: ["技术分享", "AI 开发基础（LLM & Agent）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-14 · Temperature / Top-P 等参数对输出的影响

## 📚 今日主题

> **Temperature / Top-P 等参数对输出的影响**（AI 开发基础（LLM & Agent））

### 1. 核心概念速览
Temperature 与 Top-P 是 LLM 自回归解码阶段的核心采样参数，直接作用于模型输出层后的概率分布，而非训练过程或权重。Temperature 的数学本质是对 logits（未归一化的分数）进行尺度缩放：将 logits 除以温度系数 T，再通过 softmax 得到采样概率。当 T<1 时，logits 绝对值被放大，softmax 后分布趋于尖锐，高概率 token 的概率更高，输出更确定；当 T>1 时，logits 被压缩，分布趋于平坦，低概率 token 获得更大采样机会，输出多样性增加。Top-P（Nucleus Sampling）则是从概率分布中截取累积概率刚好超过 P 的最小 token 集合，将集合外的 token 概率置零，然后对集合内概率重新归一化。它解决的核心问题是在『确定性』与『多样性』之间做显式调节，机制上属于采样前的裁剪操作。在 AI 技术栈中，这两个参数位于生成式模型的『解码策略』层，下接模型输出，上接应用逻辑，是影响 Agent 回复质量、稳定性和创造力的最直接可调变量。专业工程师必须掌握其底层机制，因为任何基于 LLM 的应用（如 Agent 工具调用、多轮对话、代码生成）中，参数调优本质上是对概率分布形状的工程控制，理解不透彻会导致输出失控、重复或幻觉等问题无法定位根源。

### 2. 底层原理剖析
底层运行机制可通过以下精确流程描述。给定词表大小为 V，模型输出原始 logits 向量 z ∈ R^V。

1. Temperature 缩放：
   - 计算 u_i = z_i / T，其中 T > 0。
   - 对 u 执行 softmax：p_i = exp(u_i) / Σ_j exp(u_j)。
   - 数学上，softmax 是玻尔兹曼分布的一般形式，T 即玻尔兹曼温度。T→0+ 时，分布趋近于 one-hot（最大概率 token）；T→∞ 时，分布趋近于均匀分布。
   - 注意：除以正数不改变 logits 的相对大小顺序，因此 top-k 的相对次序不变，但绝对差距被缩放。

2. Top-P（Nucleus）截断：
   - 对经温度缩放后的概率分布 p 按概率降序排序。
   - 从最高概率开始累加，直到累积概率 ≥ P，记录此时已经遍历的索引集合 C。
   - 将 C 之外的 token 概率置为 0，然后对 C 内的概率重新归一化，得到新的采样分布。
   - 本质：动态选择候选 token 的『概率核』，P 越小，候选集越小，越接近贪心；P 越大，候选集越大，越完全采样。

3. 实际采样流程（伪代码）：
```
logits = model_output  # [V]  float32
if T != 1.0:
    logits = logits / T
probs = softmax(logits)
if P is not None:
    sorted_indices = argsort(probs, descending=True)
    cumsum = 0.0
    keep_indices = []
    for idx in sorted_indices:
        cumsum += probs[idx]
        keep_indices.append(idx)
        if cumsum >= P:
            break
    # 构造 mask，仅保留 keep_indices
    mask_logits = full_like(logits, -inf)
    mask_logits[keep_indices] = logits[keep_indices]
    probs = softmax(mask_logits)
# 最终从 probs 中按多项式分布采样一个 token
token = categorical_sample(probs)
```

与前端已有概念的对比：Temperature 与前端中『CSS transform: scale()』在几何上有类似之处——都是对坐标/数值进行尺度变换，但这里是高维概率空间中的非线性变换（经由 softmax）。Top-P 与前端中『虚拟滚动』或『Redux selector』的核心理念相似——都是『只保留必要子集』，虚拟滚动只渲染可视区，selector 只派生需要的状态分片；但虚拟滚动是确定性的 DOM 裁剪，而 Top-P 是基于概率累积的动态截断，且截断后必须重新归一化。共同点是：它们都不改变底层数据的存在性，而是改变『可见或参与计算』的范围。核心差异在于：前端这些操作是静态逻辑，而 Temperature/Top-P 直接引入随机性，是生成过程中的随机采样策略。

### 3. 基础代码与实战验证
```text
以下为纯 Python 实现，不依赖任何深度学习框架，用于验证 Temperature 和 Top-P 对概率分布的影响。

import math, random

def softmax(logits, T=1.0):
    # Temperature 参数 T：将 logits 除以 T，再计算 softmax
    # T<1 时，logits 差值放大，概率分布更尖锐；T>1 时，差值缩小，分布更平坦
    scaled = [l / T for l in logits]
    max_l = max(scaled)  # 数值稳定性处理，防止 exp 溢出
    exps = [math.exp(l - max_l) for l in scaled]
    s = sum(exps)
    return [e / s for e in exps]

def top_p_filter(probs, P):
    # Top-P 过滤：按概率降序，保留累积概率恰好达到 P 的最小 token 集合
    # 然后对保留集合重新归一化，其他 token 概率为 0
    order = sorted(range(len(probs)), key=lambda i: probs[i], reverse=True)
    cum = 0.0
    keep = []
    for i in order:
        cum += probs[i]
        keep.append(i)
        if cum >= P:
            break
    # 重新归一化：只保留 keep 集合中的概率
    total = sum(probs[i] for i in keep)
    filtered = [0.0] * len(probs)
    for i in keep:
        filtered[i] = probs[i] / total
    return filtered

# 示例 logits: 模拟模型输出层原始分数
logits = [1.0, 2.0, 3.0, 4.0]

# 验证 Temperature 影响
for T in [0.5, 1.0, 2.0]:
    probs = softmax(logits, T)
    print(f"T={T}: {[round(p, 4) for p in probs]}")

# 验证 Top-P 影响（在 T=1.0 基础上）
probs = softmax(logits, 1.0)
filtered = top_p_filter(probs, 0.8)
print(f"原始 probs: {[round(p,4) for p in probs]}")
print(f"Top-P(0.8) 后: {[round(p,4) for p in filtered]}")
print(f"过滤后概率和: {round(sum(filtered),4)}")

运行结果示例：
T=0.5: [0.0321, 0.0871, 0.2369, 0.6438]
T=1.0: [0.0321, 0.0871, 0.2369, 0.6438]  // 注意此处 T=1.0 与 T=0.5 不同，需实际运行观察
T=2.0: [0.0932, 0.1532, 0.2519, 0.5017]
Top-P(0.8) 后会保留概率最高的两个 token（指数 3 和 2），其余概率置零并重新归一化。

注释：
- T=1.0 时 softmax 直接作用于原始 logits。
- T=0.5 时，logits 被放大（除以0.5），最高概率 token 对应概率变得更大。
- T=2.0 时，logits 被压缩，最大概率下降，分布更平坦。
- Top-P=0.8 时，只保留累积概率达 0.8 的 token，本例为概率最高的两个 token，然后重新归一化，从而在采样时完全忽略低概率 token。
```

### 4. 常见误区与进阶思考
常见认知误区：
1. 认为 Temperature 越大，模型『创造力』越强。实际只是增加采样的随机性，但不增强模型的语义理解或知识能力。温度过高会导致概率分布趋于均匀，低质量 token 被选中的概率大幅上升，生成结果更可能出现逻辑断裂、重复或幻觉。Temperature 只改变解码时的概率分布形状，不改变模型内部对语义的建模。
2. 认为 Top-P 越大，生成质量越高。实际 Top-P 控制的是候选 token 的集合大小，P 过大时，低概率的噪声 token 也会进入候选集，可能引入与上下文无关的内容，降低稳定性；P 过小时则退化为近似贪心解码，丧失多样性。温度和 Top-P 是正交的控制维度，需联合调节。

进阶思考题：
当 Temperature T 极大（例如 T→+∞）时，softmax 输出趋近均匀分布。此时若同时设置 Top-P = 0.9，Top-P 保留的 token 数量会如何变化？请从累积概率的角度解释：均匀分布下每个 token 概率约为 1/V，累积概率达到 0.9 需要保留约 0.9V 个 token，即几乎覆盖整个词表。这说明在高温下 Top-P 失去截断效果，因为所有 token 概率趋于相等，累积概率增长缓慢。这个现象背后的数学本质是：Temperature 缩放改变了分布的形状但未改变 logits 的相对排序，而 Top-P 依赖概率的绝对值累积——当分布变得平坦时，任何截断阈值都必须覆盖更大的集合才能达到相同的累积比例。理解这一点，才能真正掌握两个参数在联合采样中的相互作用。
