---
title: "每日基础技术总结 · 2026-07-01 · FlashAttention 的 IO 感知分块算法"
date: 2026-07-01 08:00:00
categories: [技术分享]
tags: ["技术分享", "AI 开发基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-01 · FlashAttention 的 IO 感知分块算法

## 📚 今日主题

> **FlashAttention 的 IO 感知分块算法**（AI 开发基础）

### 1. 核心概念速览
FlashAttention 是一种 IO 感知的精确注意力计算方法，其本质是将标准注意力计算中的 QK^T、Softmax、PV 三步融合为单次 kernel 执行，通过分块（tiling）将计算切分为可在 SRAM 中驻留的块，避免将 N×N 注意力矩阵写回 HBM。它解决的问题是注意力计算在长序列下受 HBM 带宽限制，而非算力限制；机制是利用在线 Softmax（online softmax）算法在分块内维护局部最大值与归一化因子，从而在无需访问完整行的情况下得到与标准注意力数值等价的结果。它在计算机体系结构中处于『计算访存比优化』与『GPU 内存层次结构利用』的交叉点，是 AI 系统性能工程的核心范式。专业工程师必须掌握它，因为它揭示了算法设计如何与硬件内存层次交互，是设计大规模模型训练/推理系统的底层能力基石。

### 2. 底层原理剖析
标准注意力计算流程：S = QK^T（形状 N×N），P = softmax(S)（逐行归一化），O = PV。每一步都会将中间矩阵 S 和 P 写回 HBM，产生 O(N^2) 的访存开销。FlashAttention 的核心机制是分块 + 在线 Softmax。

假设 Q、K、V 按行分块，块大小为 B_r（Q 的行块）和 B_c（K/V 的列块）。外层循环遍历 K/V 块，内层循环遍历 Q 块。对于每个 Q 块 Q_i，维护一个累加器 O_i（形状 B_r×d）以及行统计量：当前最大值 m_i（B_r 向量）和当前归一化和 l_i（B_r 向量）。

处理第 j 个 K/V 块时：
1. 从 HBM 加载 K_j, V_j 到 SRAM。
2. 计算 S_ij = Q_i K_j^T（SRAM 中，形状 B_r×B_c）。
3. 对 S_ij 的每一行，计算块内最大值 m_ij = rowmax(S_ij)。
4. 计算新的全局最大值 m_new = max(m_i, m_ij)。
5. 更新归一化因子：l_i = l_i * exp(m_i - m_new) + rowsum(exp(S_ij - m_new))。
6. 重新缩放 O_i：O_i = O_i * (exp(m_i - m_new) / (l_i_old / l_i_new)) —— 实际实现中通常用因子 exp(m_i - m_new) 先缩放，再除以 l_new。
7. 计算 P_ij = exp(S_ij - m_new)（在 SRAM 中），然后累加 O_i += P_ij V_j。
8. 最终在完成所有块后，将 O_i 除以 l_i 得到正确输出。

关键点是：不需要保存完整的 S 和 P，只需要在每个块内计算并立即使用，从而将 HBM 访问量从 O(N^2) 降至 O(N^2 * d / M) 其中 M 是 SRAM 大小，实际中接近线性。

与前端概念的对比：类似『流式处理』与『全量数据中转』的区别。前端中，如果使用 Array.prototype.map 对数组做变换，中间数组会完整驻留内存；而使用迭代器/生成器（Generator）逐个处理元素，内存占用为 O(1)。FlashAttention 的分块相当于生成器模式，但更复杂——因为 softmax 的归一化因子依赖于全行最大值，不能简单地流式求和，因此必须使用在线 Softmax 的数学技巧来保持数值一致性。另一个类比是前端中实现虚拟列表：不渲染所有 DOM 节点，只渲染可视区域，并在滚动时动态更新；FlashAttention 也是只计算 SRAM 能容纳的块，而不用物化整个 N×N 矩阵。

### 3. 基础代码与实战验证
以下为文字化伪代码（不依赖具体框架，描述 CUDA kernel 的核心逻辑）：

```
// 输入: Q, K, V (N×d), 块大小 B_r, B_c, SRAM 大小约束
// 输出: O (N×d)
for i in range(0, N, B_r):  // 外层：Q 块
    O_i = zeros(B_r, d)     // 累加器，驻留 SRAM
    m_i = full(B_r, -inf)   // 当前行最大值
    l_i = zeros(B_r)        // 当前行归一化和
    for j in range(0, N, B_c):  // 内层：K/V 块
        Q_i = load(Q[i:i+B_r])        // 从 HBM 加载到 SRAM
        K_j = load(K[j:j+B_c])        // 从 HBM 加载到 SRAM
        V_j = load(V[j:j+B_c])        // 从 HBM 加载到 SRAM

        S_ij = Q_i @ K_j.T            // 形状 B_r×B_c，在 SRAM 中计算
        m_ij = rowmax(S_ij)           // 每行的局部最大值
        m_new = elementwise_max(m_i, m_ij)  // 更新全局行最大值
        P_ij = exp(S_ij - m_new)      // 在 SRAM 中计算，避免写回 HBM
        l_i = l_i * exp(m_i - m_new) + rowsum(P_ij)  // 更新归一化和
        // 缩放之前累加的 O_i，消除旧最大值影响
        O_i = O_i * exp(m_i - m_new)
        O_i += P_ij @ V_j             // 累加贡献，形状 B_r×d
        m_i = m_new                   // 更新当前最大值
    O_i = O_i / l_i[:, None]          // 最终行归一化，得到正确输出
    write O_i to HBM
```

注释：
- 第 3 行：初始化为 -inf，保证第一次块计算的 m_new 等于 m_ij。
- 第 9 行：exp(S_ij - m_new) 使用全局新最大值，确保数值稳定性。
- 第 11 行：O_i 乘以 exp(m_i - m_new) 是因为之前累加时用的是旧最大值 m_i 进行指数衰减，现在需要调整到新最大值尺度。
- 第 13 行：最终除以 l_i，完成 softmax 的归一化。

整个过程所有中间矩阵 S_ij 和 P_ij 都在 SRAM 中产生并立即消费，不落 HBM，从而减少 O(N^2) 的访存。

### 4. 常见误区与进阶思考
常见误区 1：认为 FlashAttention 是近似注意力或稀疏注意力。实际上它是精确注意力，结果与标准 softmax 注意力数值等价（在浮点误差范围内）。它只是改变了计算顺序和访存方式，没有忽略任何注意力权重。
常见误区 2：认为分块大小越大越好。实际上分块大小受限于 SRAM 容量，而且需要容纳 Q_i, K_j, V_j, S_ij, P_ij, O_i 等多个矩阵；块过大导致 SRAM 溢出或 occupancy 下降，反而降低性能。块大小的选择需要根据具体 GPU 的 SRAM 大小和寄存器文件进行调优。

思考题：在线 Softmax 中，为什么必须在每个块内计算 P_ij 时使用 m_new 而不是 m_i？如果直接使用 m_i 计算 exp(S_ij - m_i)，在累加后最终除以 l_i 会出现什么数值问题？请从数学等价性和浮点精度的角度分析。
