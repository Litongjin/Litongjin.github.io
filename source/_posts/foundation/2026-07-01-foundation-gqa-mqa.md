---
title: "每日基础技术总结 · 2026-07-01 · 分组查询注意力 GQA 与多查询注意力 MQA"
date: 2026-07-01 08:00:00
categories: [技术分享]
tags: ["技术分享", "AI 开发基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-07-01 · 分组查询注意力 GQA 与多查询注意力 MQA

## 📚 今日主题

> **分组查询注意力 GQA 与多查询注意力 MQA**（AI 开发基础）

### 1. 核心概念速览
GQA（Grouped-Query Attention）与 MQA（Multi-Query Attention）是 Transformer 自注意力机制中用于压缩 KV 缓存、降低自回归解码内存带宽的变体。MQA 的核心是让所有注意力头共享同一组 K、V 投影，仅保留独立的 Q 投影；GQA 则是将注意力头划分为若干组，组内共享 K、V 投影，是 MHA（每头独立 K/V）与 MQA（全局共享 K/V）的插值。机制上，输入 X 通过三个线性投影得到 Q、K、V，其中 K/V 的投影输出维度从 H 个头降为 G 组（MQA 中 G=1），然后在每个头内部执行缩放点积注意力：head_i = softmax(Q_i * K_{group(i)}^T / sqrt(d_k)) * V_{group(i)}。该机制解决的核心问题是：LLM 推理时 KV 缓存大小与内存带宽是吞吐和延迟的主要瓶颈，尤其长序列与大批次场景下，缓存访问带宽远超计算量。GQA/MQA 位于 Transformer 架构与推理引擎的交汇点，直接影响 KV 缓存容量、显存占用和 batch size 上限，是专业工程师设计服务端推理系统、理解模型结构时必须掌握的底层优化手段。

### 2. 底层原理剖析
以 MHA 为基准：MHA 中 Q/K/V 投影各自输出 H*d_head 维，缓存每个 token 的 K/V 总量为 2*H*d_head 个浮点数。GQA 将 H 个头分成 G 组，K/V 投影输出 G*d_head 维，缓存量降为 2*G*d_head。MQA 是 G=1 的极端情况。

伪代码描述前向过程：
输入 X: [batch, seq, d_model]
Q = X @ W_q   # W_q: [d_model, H * d_head] -> [B, seq, H, d_head]
K = X @ W_k   # W_k: [d_model, G * d_head] -> [B, seq, G, d_head]
V = X @ W_v   # W_v: [d_model, G * d_head] -> [B, seq, G, d_head]

将 Q 重塑为 [B, G, H/G, seq, d_head]（即把 H 个头按顺序均分到 G 组）；
将 K、V 重塑为 [B, G, seq, d_head]；
在注意力计算中，将 K/V 扩展为 [B, G, 1, seq, d_head]，与组内 H/G 个 Q 头广播对齐；
计算 scores = Q @ K^T / sqrt(d_head) -> [B, G, H/G, seq, seq]；
attn = softmax(scores, dim=-1)；
out = attn @ V -> [B, G, H/G, seq, d_head]；
out 重塑回 [B, seq, H*d_head]，再过输出投影 W_o。

底层本质是张量形状与广播：K/V 的物理存储只有 G 份，多个 Q 头通过广播逻辑上访问同一份数据，不产生额外拷贝。与前端知识对比：类似 TypeScript 接口与 Java 接口的区别——TS 接口是编译期的结构化约束，运行时不存在；而 GQA 的 KV 共享是运行时的数据依赖关系，由投影矩阵的输出维度直接决定，是一种显式的内存布局设计。更准确地说，它类似多个 React 组件订阅同一个不可变 store，所有读取者看到同一份内存状态，但 GQA 没有任何调度开销，只是纯线性代数上的视图共享。

### 3. 基础代码与实战验证
```text
def gqa_attention(X, W_q, W_k, W_v, W_o, num_heads, num_groups):
    # X: [B, T, C] 输入序列，T 为序列长度，C 为模型维度
    B, T, C = X.shape
    H = num_heads          # 注意力头总数
    G = num_groups         # KV 分组数，MQA 时 G=1
    D = C // H             # 每个头的维度

    # Q 保持多头：线性投影后 reshape 为 [B, T, H, D]
    Q = X @ W_q            # W_q: [C, H*D]
    Q = Q.view(B, T, H, D)

    # K/V 只投影到 G 组，而不是 H 组，这是 GQA 的核心：
    # 投影矩阵的输出维度决定了物理上只有 G 份 KV 缓存
    K = X @ W_k            # W_k: [C, G*D]
    K = K.view(B, T, G, D)
    V = X @ W_v            # W_v: [C, G*D]
    V = V.view(B, T, G, D)

    # 将 Q 重塑为 [B, G, H//G, T, D]：把 H 个头按顺序均分到 G 组
    Q = Q.view(B, G, H // G, T, D)

    # K 的维度为 [B, G, D, T]：转置使 seq 维度在最后，以便矩阵乘法
    # unsqueeze(2) 得到 [B, G, 1, D, T]，广播到 Q 的 H//G 个头
    scores = torch.matmul(Q, K.unsqueeze(2).transpose(-1, -2)) / (D ** 0.5)
    # scores 形状: [B, G, H//G, T, T]；最后两个维度是 query 位置与 key 位置

    # 对最后一个维度（key 序列）做 softmax
    attn = torch.softmax(scores, dim=-1)

    # V unsqueeze(2) 得到 [B, G, 1, T, D]，与 attn 的组内头维度广播对齐
    out = torch.matmul(attn, V.unsqueeze(2))
    # out 形状: [B, G, H//G, T, D]

    # 恢复为 [B, T, H, D]，再拼接所有头并通过输出投影
    out = out.view(B, T, H * D)
    return out @ W_o        # W_o: [H*D, C]

# 关键点：K/V 在物理上只有 G 份，但通过广播逻辑上服务所有 H 个头；
# 当 G=H 时退化为 MHA，当 G=1 时退化为 MQA。
```

### 4. 常见误区与进阶思考
误区一：把 GQA/MQA 当成 KV 缓存压缩或量化。实际上它们不是对 KV 张量做后处理压缩，而是从模型结构上让多个注意力头共享同一组 K/V 投影，从而在源头减少缓存条目。共享发生在投影层，不是缓存管理层的技巧；量化/剪枝不改变投影输出维度，而 GQA/MQA 改变的是 W_k/W_v 的输出维度。

误区二：认为 GQA 可以无损替换 MHA 或质量完全不变。减少 KV 头数会削弱模型对不同注意力头差异化读取信息的能力，尤其在需要精细位置区分或多种关系建模时会有可测量的质量下降。更隐蔽的是，不能简单地把 MHA 权重直接改名成 GQA 使用；GQA 的 W_k/W_v 输出维度是 G*D 而非 H*D，必须通过预训练或专门的投影合并/蒸馏才能转换。

思考题：假设你有一个训练好的 MHA 模型，W_k 的形状为 [C, H*D]。现在要求在不重新训练、不改变参数形状的前提下，仅修改前向代码中的索引或掩码，让所有头共享第一份 K/V。这样能实现 GQA 的推理加速吗？为什么？——这直接检验你是否理解“缓存容量由投影输出维度决定”这一本质。若 W_k 仍输出 H*D，物理缓存仍是 H 份，即使注意力只读第一份，内存带宽和缓存占用并未减少，真正的 GQA 必须从权重矩阵的输出维度上收缩 K/V 的物理数量。理解这一点，才算真正看穿 GQA 的底层机制。
