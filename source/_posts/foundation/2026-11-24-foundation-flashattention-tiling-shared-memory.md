---
title: "每日基础技术总结 · 2026-11-24 · FlashAttention 的 Tiling 策略与 Shared Memory 复用机制"
date: 2026-11-24 08:00:00
categories: [技术分享]
tags: ["技术分享", "AI 开发基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-24 · FlashAttention 的 Tiling 策略与 Shared Memory 复用机制

## 📚 今日主题

> **FlashAttention 的 Tiling 策略与 Shared Memory 复用机制**（AI 开发基础）

### 1. 核心概念速览
FlashAttention 是基于 GPU 内存层次设计的精确注意力算法，其本质是把 Attention 从『先物化 N×N 分数矩阵，再 softmax，再矩阵乘』的 HBM 密集型流程，改造为按 tile 在 shared memory/SRAM 中完成 QK^T、softmax 归约与 PV 累加的 IO-aware 流式计算。它解决的核心问题不是减少 FLOPs，而是消除 O(N^2) 中间张量读写与 HBM 带宽瓶颈，并通过 online softmax 的 running max/sum 保证分块计算与标准 softmax 数学等价。它位于 AI 系统栈的算子层与硬件内存层次之间，是理解 LLM 训练/推理性能、长上下文、KV cache、WebGPU/CUDA compute kernel 优化的基础。专业工程师必须掌握它，因为现代 Transformer 的性能瓶颈已从算力转向内存带宽，不懂 tiling 与 shared memory 就无法理解推理吞吐、显存占用与 kernel 行为。

### 2. 底层原理剖析
标准 Attention：S=QK^T，P=softmax(S)，O=PV。朴素实现会把 S、P 物化到 HBM，导致 O(N^2) 显存与读写。FlashAttention 将 Q、K、V 切成 Br×d、Bc×d 的 tile，只把当前 tile 加载到 shared memory，完成局部 S_ij=Q_iK_j^T、局部 exp、局部 PV 累加。由于 softmax 需要全局 max 与 sum，算法维护每行统计量 m_i、l_i 与未归一化输出 O_i。每处理一个新的 K/V tile，就执行：m_new=max(m_old,rowmax(S_ij)); P_ij=exp(S_ij-m_new); l_new=exp(m_old-m_new)l_old+rowsum(P_ij); O_new=exp(m_old-m_new)O_old+P_ijV_j。旧的 O 和 l 通过 exp(m_old-m_new) 重缩放，保证数学等价。Shared memory 复用体现在：Q tile、K tile、V tile、S tile、P tile 的生命周期被严格限制在一次 tile 计算内；同一块 SMEM 通过阶段划分与 swizzle 布局反复承载不同矩阵，避免把 S/P 写回 HBM，并降低 bank conflict。
与前端已有知识对比：它类似虚拟滚动中的『只渲染可见区块』，但本质不是 UI 懒加载，而是带数值稳定性约束的流式归约；如果熟悉 WebGPU compute shader，workgroup memory 就是同一类 programmer-managed shared memory，FlashAttention 相当于在 shader 中显式控制 tile 加载、同步、局部归约与写回，而不是把所有中间结果放进全局 buffer。与 TS/Java 接口差异不同，那里比较的是类型系统语义；这里比较的是内存层次与执行模型：前端通常面对 Cache API、IndexedDB、主线程内存，而 GPU kernel 面对的是 HBM、L2、shared memory、register 的带宽/容量金字塔。

### 3. 基础代码与实战验证
```text
# 验证 FlashAttention 的 tiling 与 online softmax，不涉及真实 GPU；核心思想与 kernel 内循环一致。
import numpy as np

def flash_attention_tiled(Q, K, V, Br, Bc):
    N, d = Q.shape
    O = np.zeros_like(Q)              # 未归一化输出累加器，对应 register/shared memory 中的局部结果
    m = np.full((N, 1), -np.inf)      # 每行 running max，保证 exp 数值稳定
    l = np.zeros((N, 1))              # 每行 running sum，等价于 softmax 分母

    # 外层遍历 K/V tile：每次只希望当前 K/V 块驻留在高速内存
    for j0 in range(0, N, Bc):
        Kj = K[j0:j0 + Bc]
        Vj = V[j0:j0 + Bc]

        # 内层遍历 Q tile：每个 Q 块独立维护自己的 m、l、O
        for i0 in range(0, N, Br):
            Qi = Q[i0:i0 + Br]

            # 局部 S = Qi @ Kj^T；真实 kernel 中该矩阵只驻留 shared memory/registers
            S = Qi @ Kj.T

            m_tile = S.max(axis=1, keepdims=True)
            m_new = np.maximum(m[i0:i0 + Br], m_tile)

            # alpha 是旧统计量在新 max 下的重缩放因子；这是 online softmax 的关键
            alpha = np.exp(m[i0:i0 + Br] - m_new)

            P = np.exp(S - m_new)     # 局部注意力权重，未除以全局 sum
            l_new = alpha * l[i0:i0 + Br] + P.sum(axis=1, keepdims=True)

            # 旧 O 先重缩放，再累加当前 tile 的 P @ Vj
            O[i0:i0 + Br] = alpha * O[i0:i0 + Br] + P @ Vj

            m[i0:i0 + Br] = m_new
            l[i0:i0 + Br] = l_new

    # 最终归一化，得到与标准 Attention 数学等价的结果（浮点误差范围内）
    return O / l
```

### 4. 常见误区与进阶思考
误区一：把 FlashAttention 理解为近似注意力、稀疏注意力或跳过部分 QK^T。它的目标不是改变数学定义，而是改变内存访问模式；输出在浮点误差范围内等价于标准 softmax attention。
误区二：把 tiling 简单等同于矩阵乘法分块。Attention 的难点在 softmax 是跨列全局归约，若没有 running max、running sum 和对历史 O/l 的 rescale，分块结果会破坏归一化语义。
进阶思考：在 shared memory 容量固定为 S、head_dim 为 d 时，若要同时容纳 Q、K、V 和中间 S 的 tile，Br/Bc 应满足什么约束？进一步，为什么外层遍历 K/V、内层遍历 Q 会改变 O 的写回频率，而 FlashAttention-2 又为何通过调整并行维度与循环顺序来提升 SM occupancy 和减少 non-matmul 指令？
