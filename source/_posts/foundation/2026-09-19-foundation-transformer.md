---
title: "每日基础技术总结 · 2026-09-19 · Transformer 架构简述"
date: 2026-09-19 07:02:03
categories: [技术分享]
tags: ["技术分享", "AI 开发基础（LLM & Agent）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-19 · Transformer 架构简述

## 📚 今日主题

> **Transformer 架构简述**（AI 开发基础（LLM & Agent））

### 1. 核心概念速览
**定义**：Transformer 是 Vaswani 等人于 2017 年提出的序列建模架构，其核心主张是彻底移除循环（recurrence）与卷积（convolution），仅用注意力机制（Attention）与逐位置前馈网络（Position-wise FFN）堆叠成编码器/解码器，并在每个子层外包残差连接与 LayerNorm。形式化地，一个 Transformer block 是如下函数：X ∈ R^{T×d_model} → X' ∈ R^{T×d_model}，由 MHSA 子层与 FFN 子层各叠加一次残差构成。当前所有主流 LLM（GPT / LLaMA / Qwen / Claude 等）均为其 Decoder-only 变体。

**它解决的问题**：
1. RNN/LSTM 存在时序依赖，第 t 步的计算必须等待第 t-1 步，无法在时间维并行；同时梯度沿时间步连乘回传，长程依赖呈指数衰减（梯度消失/爆炸）。
2. CNN 的感受野受 kernel size 与层数限制，两个相距 T 的位置需要 O(T/k) 层才能建立联系，路径长度随距离线性增长。
3. Transformer 通过一次 QK^T 让任意两个位置之间建立直接连接，路径长度压缩到 O(1)，代价是时间与空间复杂度变为 O(T²·d_model)，即序列长度的二次方。

**本质**：Transformer 在序列维上实现了一次「权重由输入动态生成的全连接」。注意力权重不是可学习参数，而是当前输入的 Q、K 点积经 softmax 归一化后的函数输出，即 hypernetwork / dynamic filter 结构。对比之下，CNN 的卷积核是训练后固定的参数；Transformer 的「核」在每个 token、每个样本上都不相同。这使得它无法被表示为任何固定权重的线性算子。

**在计算机/AI 体系中的位置**：它是从「符号化的、离散的、不可微的」传统 NLP 流水线（分词 → 词性 → 句法 → 语义）向「端到端的、连续的、可微的」表示学习范式的分水岭。上游承接词嵌入与张量代数，下游直接决定：推理引擎的显存布局（KV cache）、训练并行策略（张量并行切分的就是 W_Q/W_K/W_V/W_O/W_1/W_2）、微调方法的作用位置（LoRA 注入的正是这些投影矩阵）、以及上下文窗口的成本模型（token 数与 T² 的关系）。

**为什么必须掌握**：Agent / RAG / 长上下文工程中所有「能用/不能用」「贵/不贵」「为什么退化」的工程结论，都是这个架构的数学结构直接推出来的，而不是某家厂商的实现细节。例如：为什么流式输出首 token 延迟（TTFT）与每 token 延迟（TPOT）是两个独立的优化目标——因为它们分别对应 prefill 阶段的 O(T²) 矩阵乘与 decode 阶段的 O(T) 访存带宽瓶颈。不理解这一点，后续所有推理优化都只能靠试错。

### 2. 底层原理剖析
**数据流（单层，Pre-LN 变体）**：

```
输入 X (T, d_model)
  ① 位置信息注入：X = X + PE   或   X 保持不变、在 Q/K 上做 RoPE 旋转
  ② Y = LayerNorm(X)
     Q = Y·W_Q,  K = Y·W_K,  V = Y·W_V        # W ∈ R^{d_model × d_k}，d_k = d_model/h
     S = Q·K^T / sqrt(d_k)                     # (T, T)
     S = S + M                                 # M 为因果掩码，上三角 = -inf
     A = softmax(S, dim=-1)                    # (T, T)，每行和恒为 1
     Z = A·V                                   # (T, d_v)
     X = X + Z·W_O                             # 残差
  ③ Y = LayerNorm(X)
     X = X + GELU(Y·W_1 + b_1)·W_2 + b_2       # 逐位置 FFN，d_ff ≈ 4·d_model
```

**逐个环节的底层机理**：

1. **为什么是 Q·K^T**：内积是向量相似度的度量。Q 的第 i 行与 K 的第 j 行做内积，得到位置 i 应当从位置 j 抽取多少信息的未归一化分数。整张 (T,T) 矩阵一次性算出全部两两关系，这是并行性的来源。
2. **为什么除以 sqrt(d_k)**：若 q、k 各维独立且均值 0、方差 1，则点积的方差等于 d_k。d_k 增大时 logits 尺度随之增大，softmax 进入饱和区（某一项接近 1，其余接近 0），梯度趋近于零，训练停滞。除以 sqrt(d_k) 把方差归一化回 1，保持 softmax 处于梯度有效的线性区间。这是一个纯数值稳定性约束，不是启发式技巧。
3. **为什么必须 softmax**：需要把实数分数转成非负、和为 1 的凸组合系数。凸组合保证输出 Z 一定落在所有 V 的凸包内，从而数值有界、可微、梯度连续。同时 softmax 的指数形式使得相似度差异被放大，具备近似 argmax 的「硬选择」能力，又保留全路径可微。
4. **为什么多头**：把 d_model 维空间切分为 h 个 d_k 维子空间，在每个子空间独立执行注意力，等于同时运行 h 组不同的动态滤波器，各自捕获不同类型的依赖关系（如局部语法共现、长距离指代等）。注意实现上是「切分」而非「独立随机初始化 h 套投影」——h 个头共享同一个 d_model 宽度的输入，只是输出被拆开，等价于对 Q/K/V 的投影矩阵做块对角化。
5. **为什么 FFN 占总参数约 2/3**：注意力本身在 V 上是线性加权（A·V 对 V 是线性的），它只做信息路由，不做特征变换。所有的非线性与绝大部分容量来自 FFN：每个位置的参数量为 2·d_model·d_ff ≈ 8·d_model²，而注意力的四个投影矩阵合计约 4·d_model²。删除 FFN 后模型表达能力坍缩为线性加权平均。
6. **复杂度**：单层前向 FLOPs ≈ 2·T·d_model + 2·T²·d_model + 16·T·d_model²（注意力两部分 + 投影与 FFN）。T² 项只在注意力中出现。
7. **训练 vs 推理**：训练时用 teacher forcing + 因果掩码，一次前向即可并行计算出全部 T 个位置的 loss（掩码保证无信息泄漏），这是 Transformer 能吃掉海量数据的根本原因；推理时自回归解码，第 t 步只依赖前 t-1 步，必须串行，且历史 K/V 可缓存复用（KV cache），显存开销 = 2 × layers × heads × d_head × T × bytes。

**与前端工程师已有概念体系的对照**：

- **软查表 vs 硬查表**：JS 的 `Map.get(key)` 是离散精确匹配、O(1)、不可微；注意力是「可微的哈希查找」——query 与全部 key 计算连续相似度，用归一化后的权重读取全部 value 并加权求和。键值不是预先注册的，而是由输入经线性投影即时生成的。这是符号计算与连续表示的根本分野。
- **并行度模型**：RNN 等价于一个无法并行化的 `reduce` 链（每一步的输入依赖上一步输出）；Transformer 等价于对数组做 `map`，但每个位置的映射函数不同，且该函数由一次全局 `scan`（注意力）先算出上下文决定。
- **置换等变 vs 序列有序**：JS 数组天生有序；Transformer 若不加位置编码，整个 block 对输入是置换等变的——交换输入任意两行，输出只是相应交换同样的两行（原理部分代码会验证）。顺序信息完全靠位置编码注入，这是它与所有有序数据结构的最大差异。
- **动态卷积核 vs 固定卷积核**：CNN 的 kernel 是训练后固化的常量张量；注意力在序列维上等价于一个 kernel size = T、且权重由输入实时生成的一维卷积。因此参数量与序列长度无关，但计算量随 T 二次增长。
- **shape 契约 vs TS 接口**：TS 接口是编译期检查、结构化类型、运行时被完全擦除；注意力中的维度约束（如 (T,d) @ (d,d) → (T,d)、残差要求输出 shape 与输入严格一致）只存在于运行时，类型系统无法表达，只能靠断言或框架的 shape 校验。把张量维度当作「接口契约」来理解，是调试 LLM 代码的核心习惯。

### 3. 基础代码与实战验证
以下为纯 NumPy 实现（不依赖任何深度学习框架），可直接运行验证因果性、凸组合性与置换等变性三个核心性质。

```
import numpy as np
np.random.seed(0)

def softmax(x, axis=-1):
    # 减去逐行最大值：仅做数值平移，softmax 结果数学上不变，但避免 exp 上溢为 inf
    x = x - x.max(axis=axis, keepdims=True)
    e = np.exp(x)
    return e / e.sum(axis=axis, keepdims=True)

def layer_norm(x, g, b, eps=1e-5):
    # 沿最后一维（特征维）归一化，与 batch 维、序列维完全无关；这是与 BatchNorm 的本质区别
    mu = x.mean(-1, keepdims=True)
    var = x.var(-1, keepdims=True)
    return (x - mu) / np.sqrt(var + eps) * g + b

def mha(x, p, h, causal=True):
    # x: (T, d_model)；h 为头数；d_k = d_model // h
    T, D = x.shape
    dk = D // h
    # 1. Q/K/V 共享同一输入 x 做线性投影，这就是 'self' 的含义
    Q = x @ p['Wq']                       # (T, D)
    K = x @ p['Wk']
    V = x @ p['Wv']
    # 2. 切分为 h 个头：把 d_model 维拆成 (h, dk)，头与头之间无参数共享、无交互
    Q = Q.reshape(T, h, dk).transpose(1, 0, 2)   # (h, T, dk)
    K = K.reshape(T, h, dk).transpose(1, 0, 2)
    V = V.reshape(T, h, dk).transpose(1, 0, 2)
    # 3. 缩放点积：每个元素是 query i 与 key j 在 dk 维上的内积
    #    除以 sqrt(dk) 把点积方差从 dk 归一化回 1，否则 softmax 饱和、梯度趋零
    S = Q @ K.transpose(0, 2, 1) / np.sqrt(dk)   # (h, T, T)
    # 4. 因果掩码：位置 i 只能看见 j <= i，上三角置 -1e9，softmax 后该处权重精确为 0
    if causal:
        mask = np.tril(np.ones((T, T), dtype=bool))
        S = np.where(mask, S, -1e9)
    A = softmax(S, axis=-1)               # (h, T, T)，每行和为 1，是凸组合系数
    Z = A @ V                             # (h, T, dk)，对所有 V 做加权聚合
    Z = Z.transpose(1, 0, 2).reshape(T, D)  # 多头拼接回 d_model
    return Z @ p['Wo'], A                 # 输出投影，回到 d_model 空间以配合残差

def block(x, p, h, causal=True):
    # Pre-LN：先归一化再进子层，残差直连，梯度可无损回传到浅层（Post-LN 需 warmup）
    x = x + mha(layer_norm(x, p['g1'], p['b1']), p, h, causal)[0]
    y = layer_norm(x, p['g2'], p['b2'])
    # FFN 逐位置独立：等价于 kernel=1 的两次卷积，提供全部非线性与约 2/3 参数量
    x = x + np.maximum(0, y @ p['W1'] + p['b1f']) @ p['W2'] + p['b2f']
    return x

# ---- 初始化参数 ----
T, D, h = 4, 8, 2
p = {
    'Wq': np.random.randn(D, D) * 0.1, 'Wk': np.random.randn(D, D) * 0.1,
    'Wv': np.random.randn(D, D) * 0.1, 'Wo': np.random.randn(D, D) * 0.1,
    'W1': np.random.randn(D, 4 * D) * 0.1, 'b1f': np.zeros(4 * D),
    'W2': np.random.randn(4 * D, D) * 0.1, 'b2f': np.zeros(D),
    'g1': np.ones(D), 'b1': np.zeros(D),
    'g2': np.ones(D), 'b2': np.zeros(D),
}

x = np.random.randn(T, D)
y = block(x, p, h)
A = mha(layer_norm(x, p['g1'], p['b1']), p, h)[1]

# 验证 1：残差要求输入输出 shape 严格一致，否则加法会广播出错
assert y.shape == x.shape == (T, D)

# 验证 2：注意力每行是凸组合系数（非负、和为 1），故 Z 必落在 V 的凸包内，输出有界
assert np.allclose(A.sum(-1), 1.0, atol=1e-6)
assert (A >= 0).all()

# 验证 3：因果性。改动 t = T-1 的输入，前 T-1 个位置的输出必须逐位不变
x2 = x.copy()
x2[-1] += 5.0
assert np.allclose(block(x, p, h)[:T-1], block(x2, p, h)[:T-1])

# 验证 4：置换等变性。关闭因果掩码后，交换输入行序，输出精确等于交换同样行序的输出
#（因为无位置编码，整个 block 沿 T 维没有任何顺序感知能力，等价于对 Set 操作）
idx = np.array([2, 0, 3, 1])
assert np.allclose(block(x[idx], p, h, causal=False), block(x, p, h, causal=False)[idx])
print('all checks passed')
```

关键点回顾：验证 3 证明掩码在数学上切断了 i→j (j>i) 的信息通路，而不是「近似地降低权重」；验证 4 证明顺序信息 100% 由位置编码承担，架构本身对序列顺序完全无感——这也解释了为什么去掉位置编码后会退化成词袋模型。

### 4. 常见误区与进阶思考
**误区一：把注意力权重当作可解释性依据。** A = softmax(QK^T/√d_k) 只是前向计算中的一个中间张量。第一，A 的数值大小不代表因果贡献——当 A[i][j] 较小但 V[j] 的模长很大时，其对 Z[i] 的实际贡献可能远超一个权重更大但值向量接近零的位置；第二，多层级联后，深层 token 已经混合了浅层的全部上下文，单条注意力边的归因意义被彻底稀释；第三，残差连接提供了绕过任何 attention 边的梯度通路。大量消融与梯度归因实验表明，注意力权重与「该 token 对输出的实际影响」的相关性极弱。在 Agent 调试中依赖 attention 可视化定位问题，方向是错的——应当用消融、梯度归因或因果干预。

**误区二：把 O(T²) 当作实现缺陷，期望靠更快的库或硬件解决。** O(T²) 来自 QK^T 这个全对全交互的数学结构，是模型定义的一部分，不是工程 bug。FlashAttention 通过 tiling + online softmax 把 HBM 读写从 O(T²) 降到接近 O(T)，但它不改变数学、不改变 FLOPs（仍是 O(T²·d)），改变的是访存模式，因此在长序列、显存带宽受限的场景收益巨大，而在短序列上收益有限。真正降低复杂度必须改数学：线性注意力、稀疏/局部注意力、状态空间模型（Mamba 等）。与之相关的第二个常见误判是低估 KV cache 的显存占用：2 × layers × heads × d_head × T × bytes，在长上下文场景中它与模型权重处于同一量级甚至更高，是推理服务容量的第一约束。

**思考题**：去掉位置编码后，整个 block 满足 y(PX) = P·y(X)（置换等变）。那么请问：(a) 可学习的绝对位置编码（直接 X = X + PE）与 RoPE（在 Q、K 上按位置做旋转）在「注入顺序信息」的数学方式上有何本质区别？(b) 请从 QK^T 的表达式出发推导，为什么 RoPE 能使注意力分数只依赖相对位移 (i − j)，而绝对位置编码不能？(c) 这个性质如何解释 RoPE 具备一定的长度外推能力，而可学习绝对位置编码在超出训练长度后迅速退化？
