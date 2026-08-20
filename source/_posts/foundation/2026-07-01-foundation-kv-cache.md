---
title: "每日基础技术总结 · 2026-07-01 · KV Cache 的原理与缓存大小计算"
date: 2026-07-01 08:00:00
categories: [技术分享]
tags: ["技术分享", "AI 开发基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-01 · KV Cache 的原理与缓存大小计算

## 📚 今日主题

> **KV Cache 的原理与缓存大小计算**（AI 开发基础）

### 1. 核心概念速览
KV Cache 是 Transformer 架构在自回归解码阶段的一种运行时缓存机制，用于缓存已生成 token 对应的 Key 矩阵和 Value 矩阵，从而避免在每一步生成时重复计算所有历史 token 的 K、V。其本质是空间换时间，利用因果注意力中前缀共享性——前 t 个 token 的 K、V 在生成第 t+1 个 token 时完全不变，因此可以直接复用。它解决的是自回归逐 token 解码中重复计算导致的 O(T^2) 复杂度（每步重新计算所有历史 K、V），将总复杂度降为 O(T)（每步仅计算新增 token 的 K、V）。KV Cache 位于大模型推理优化（inference optimization）的核心位置，与算子融合、量化、投机采样并列，直接影响显存占用、吞吐量、延迟与 batch size 上限。专业工程师必须掌握它，因为任何生产级 LLM 推理服务（vLLM、TensorRT-LLM 等）的显存规划、调度策略和性能优化都建立在对 KV Cache 的精确理解上。

### 2. 底层原理剖析
Transformer 自回归解码中，每一步的注意力计算为：
  Q = X·W_q, K = X·W_k, V = X·W_v
  Attention = softmax(Q·K^T / sqrt(d)) · V
假设当前序列长度为 L，则 K、V 的维度为 [L, d]。若每步重新计算所有历史 token 的 K、V，则第 t 步计算量为 O(t·d)，总计算量为 O(T^2·d)，而 T 通常可达数千甚至数万，不可接受。
KV Cache 的核心机制：在预填充（prefill）阶段，一次性计算输入序列所有 token 的 K、V 并缓存；在解码（decode）阶段，每步只计算当前新增 token 的 Q、K、V，将新的 K、V 与缓存拼接后，用当前 Q 与完整的 K^T 做点积，再与完整的 V 相乘得到当前 token 的输出。关键点在于：历史 K、V 从未改变，因此可以安全复用。
与前端已有概念的对比：KV Cache 类似于前端中的「函数结果缓存」（memoization）——都是缓存中间计算结果以避免重复计算。但存在本质差异：
1. KV Cache 缓存的是有状态的张量，其生命周期与序列长度强绑定，随着生成过程线性增长，受显存上限约束；而前端 memoization 缓存的是纯函数输出，无状态，可通过依赖变化主动失效。
2. KV Cache 是推理路径上的必需组件，不是可选的优化；而 memoization 是纯优化，删除后不影响正确性。
3. KV Cache 的更新模式是「增量拼接」，而不是「覆盖替换」，这与前端状态管理中的不可变更新（如 React state）有形式相似，但目的完全不同。
另一个可对比概念是「浏览器 HTTP 缓存」：两者都通过复用历史数据减少重复计算/传输，但 HTTP 缓存有明确的失效验证机制（如 ETag），而 KV Cache 的缓存内容在序列内部永不失效，只在序列结束时释放。

### 3. 基础代码与实战验证
```text
以下为极简伪代码（使用 numpy 语义，不含框架），模拟单层单头 Transformer 的 KV Cache 工作流程。

# 维度约定：d 为隐藏层维度，L 为序列长度
W_q, W_k, W_v = 随机初始化矩阵，形状均为 [d, d]

# 预填充（prefill）：一次计算输入序列所有 token 的 K、V，并输出初始注意力结果
def prefill(tokens_emb):  # tokens_emb: [L, d]
    Q = tokens_emb @ W_q.T  # [L, d]
    K = tokens_emb @ W_k.T  # [L, d]  # 全部 K 一次性算出，存入缓存
    V = tokens_emb @ W_v.T  # [L, d]  # 全部 V 一次性算出，存入缓存
    # 注意力计算，mask 为下三角矩阵（因果掩码），保证当前位置只看到历史 token
    attn = softmax(Q @ K.T / sqrt(d) + mask) @ V  # [L, d]
    return K, V  # 缓存 K、V，供后续解码使用

# 解码单步（decode）：只计算新 token 的 K、V，并与缓存拼接
def decode_step(x_new, K_cache, V_cache):  # x_new: [1, d], K_cache: [L, d]
    q = x_new @ W_q.T  # [1, d]，新 token 的 Q
    k = x_new @ W_k.T  # [1, d]，新 token 的 K
    v = x_new @ W_v.T  # [1, d]，新 token 的 V
    # 关键：将新的 K、V 追加到缓存末尾，形成 [L+1, d]
    K_new = np.concatenate([K_cache, k], axis=0)  # 新 K 拼入缓存
    V_new = np.concatenate([V_cache, v], axis=0)  # 新 V 拼入缓存
    # 注意力计算：q 与所有 K 做点积，再与所有 V 加权求和
    attn = softmax(q @ K_new.T / sqrt(d)) @ V_new  # [1, d]
    return attn, K_new, V_new  # 输出并返回更新后的缓存

# 使用流程：
# 1. 对 prompt 调用 prefill，得到初始 K_cache, V_cache
# 2. 循环调用 decode_step，每次输入上一步输出的 embedding，生成下一个 token
# 3. 每次 decode_step 只计算 1 个 token 的 K、V，而历史 token 的 K、V 直接从缓存读取

注意：上述代码省略了 embedding、layer norm、multi-head 拆分和输出投影，仅展示 KV Cache 最本质的缓存复用逻辑。真实实现中，K、V 是按层、按头分别缓存的，且拼接发生在内存中的预分配显存块上，避免重新分配。
```

### 4. 常见误区与进阶思考
误区一：认为 KV Cache 缓存的是所有中间结果，包括 Q、注意力矩阵等。实际只缓存 K 和 V，Q 不缓存（每个新 token 的 Q 不同），注意力矩阵更不能缓存（新 Q 会重新归一化所有历史分数）。
误区二：计算缓存大小时遗漏层数、头数或精度。正确公式：总缓存字节数 = 2 × num_layers × num_heads × head_dim × seq_len × batch_size × dtype_size。其中 num_heads × head_dim = hidden_dim，因此每个 token 每层需要 2 × hidden_dim 个数值。若不乘层数或忽略精度，显存估算会严重偏小。

思考题：在自回归解码中，假设我们缓存了第 t 步的注意力概率矩阵 P_t = softmax(Q_t K_t^T / sqrt(d))。当生成第 t+1 个 token 时，能否复用 P_t 来加速计算？为什么不能？请从 softmax 的归一化性质与 Q 的更新机制出发，证明缓存注意力概率矩阵是无效的，从而理解 KV Cache 必须缓存 K、V 而不是 P 的本质原因。
