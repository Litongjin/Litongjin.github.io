---
title: "每日基础技术总结 · 2026-09-05 · 大模型推理量化：INT8/INT4 与 AWQ/GPTQ"
date: 2026-09-05 08:00:00
categories: [技术分享]
tags: ["技术分享", "AI / LLM 工程实战"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-05 · 大模型推理量化：INT8/INT4 与 AWQ/GPTQ

## 📚 今日主题

> **大模型推理量化：INT8/INT4 与 AWQ/GPTQ**（AI / LLM 工程实战）

### 1. 核心概念速览
大模型推理量化（Quantization）是将模型参数/激活从 FP16/BF16 高精度浮点映射到 INT8/INT4 等低比特整数表示的压缩与加速技术。其本质是：利用神经网络的权重与激活分布具有近似高斯/拉普拉斯形状、且对噪声有一定容忍度的特性，通过离散化信息承载位宽，减小模型体积并利用整数矩阵运算（如 INT8 GEMM）提升推理吞吐。它解决的核心问题包括：显存带宽瓶颈、显存容量不足、以及 FP16 矩阵乘法在消费级硬件上的低效与延迟。机制上分为训练后量化（PTQ）与量化感知训练（QAT），当前 LLM 主打 PTQ 路线。在整个计算机体系中，它属于“数值表示与算子优化”与“模型压缩”的交叉领域，是连接深度学习模型与实际 GPU/CPU 指令集（INT8 Tensor Core / INT4 查表 SIMD）的桥梁。专业工程师必须掌握，因为大模型部署的工程核心就是精度-成本-性能的权衡，量化直接决定推理服务能否在受限 GPU 环境落地，且其数学与算子层面的细节直接影响服务的稳定性与响应延迟。

### 2. 底层原理剖析
底层注册机制：量化本质构造一个映射 q = round(r / scale) + zero_point，其中 scale 是对应张量/通道的实步长，zero_point 是整型偏移，反量化 r ≈ scale * (q - zero_point)。GPTQ 与 AWQ 都是在找这个映射的“最优”scale 与 zero_point，并减少量化误差对输出的级联影响。

GPTQ（基于二阶近似）—— 逐层最优权重编码：
它以 Hessian 矩阵 H = 2 X^T X + λI（其中 X 是权重所在层的输入激活）近似权重扰动对最终误差的影响。对每一层，GPTQ 按列处理权重量化，每次量化一列时，利用 H 的逆信息计算对该列权重扰动的最优补偿，并把误差通过 “梯度” 分配到未量化的列上，最终所有列依次量化完毕。该过程类似“贪婪算法 + 误差回传”，使整体量化误差在 L2 范数意义下最小。

AWQ（基于激活感知）—— 按通道缩放保护显著特征：
AWQ 不迭代修正权重，而是观察激活分布的绝对值统计，识别对量化更敏感的通道（激活值幅度大的通道），然后对这些通道的权重乘以一个缩放系数 s，并同时调整激活的缩放，从而在数学上保持线性层输出不变：y = (W * s)(x / s)。量化时 W 的分布被 s 拉平，减小截断误差，而那些敏感通道也因此保留更多信息。这个 s 是每个通道一个浮点标量，通过网格搜索或梯度优化求得。

对比前端概念：Java 的接口与 TS 的接口——Java 接口是编译期强制约束且运行期存在类型信息（Type Erasure 后仍标记），TS 接口只存在于编译期，运行期消失。两者的“结构类型”与“名义类型”差异本质是静态类型系统在“具体是否存在”上的分歧。类似地，GPTQ 与 AWQ 都是“降低 weight 位宽”的方法，但一个通过“迭代补偿”在数值空间做全局误差最小化，一个通过“输入激活感知”在数据分布上做局部重要性保护，它们解决的问题域不同：GPTQ 解决“量化误差如何分摊”，AWQ 解决“哪些权重值得保护”。理解这两者的差异，就是理解“误差最小化”与“信息保护”在数值表示上的不同策略。

底层运行流程（INT8 推理的算子级逻辑）：
1. 权重在离线时完成 int8 量化，得到 quantized_weight (int8) 和 scale (float)。
2. 输入激活在运行时先用一个“动态/静态 scale”量化为 int8，假设 scale_act。
3. INT8 GEMM 计算：accumulator = sum(int8_w * int8_act)，获得 int32 整型累加结果。
4. 反量化：output = accumulator * (scale_w * scale_act)，再 cast 回 fp16 供后续层使用。

INT4 类似，但通常反量化早或使用更优化的内核（如对 4-bit 权重查表）。

### 3. 基础代码与实战验证
```text
以下为纯 Python/NumPy 风格的伪代码，演示 GPTQ 与 AWQ 的核心机制（不依赖任何深度学习框架，可直接读）。

# ---------- GPTQ 单层量化核心 ----------
import numpy as np

def gptq_quantize_layer(W, H, bits=4):
    """W: [out_features, in_features]，H: Hessian 矩阵，列级量化"""
    W = W.clone()
    Q = np.zeros_like(W)  # 量化后的权重
    qmax = 2**bits - 1
    # 取 H 的逆（Hessian 逆矩阵代表误差传播的协方差）
    Hinv = np.linalg.inv(H)
    for i in range(W.shape[1]):
        # 当前列权重
        w = W[:, i]
        # 根据 Hessian 逆的对角线计算缩放 scale（更精确：考虑阻尼）
        scale = np.abs(w).max() / qmax  # 简化为最大值缩放
        q = np.round(w / scale).astype(np.int8)  # 量化
        q = np.clip(q, -qmax, qmax)
        Q[:, i] = q
        # 计算误差
        err = (q * scale - w).reshape(-1, 1)
        # 把误差按 Hinv 的相关性分配到尚未量化的列：
        # 核心公式：delta_w = -err / Hinv[i,i] * Hinv[i, j:]  (j>i)
        # 实际实现利用 Cholesky 分解避免求逆，本质相同。
        # 这里简化：将误差沿 Hinv[i, i+1:] 方向映射到其他列
        if i + 1 < W.shape[1]:
            weight_share = Hinv[i, i+1:] / (Hinv[i, i] + 1e-8)
            W[:, i+1:] += err * weight_share  # 更新未量化列
    return Q, scale

# ---------- AWQ 按通道缩放核心 ----------
def awq_apply_scaling(W, x, s):
    """W: [out, in]，x: [in]，s: 每个输入通道的缩放因子"""
    # 缩放权重：等价于 y = Wx，先将 W 的列缩放 s，再将 x 的对应行缩小 1/s
    W_scaled = W * s.reshape(1, -1)
    x_scaled = x / s.reshape(-1, 1)
    return W_scaled, x_scaled
    # 之后 W_scaled 再做常规 per-channel 量化，误差由 s 控制。

# 实际 AWQ 中 s = (max|activation| / mean|activation|)^alpha 启发式得到，alpha 为超参。

关键点：GPTQ 是“量化-补偿-继续”的循环，误差在未量化列间流动；
AWQ 是“先缩放-再量化”的预处理，保护了激活幅度大的通道。

若要在真实 PyTorch 中验证，建议使用 GPTQ-for-LLaMA 或 AutoAWQ 集成，但上述无依赖伪代码已能表达数学本质。
```

### 4. 常见误区与进阶思考
常见误区 1：认为量化只是“损失一点精度”，于是随意选择全局 scale。实际 LLM 权重中会出现极端值（outlier），如果采用全局量化（整个张量一个 scale），极少数大值会拉大 step 导致绝大多数小值被抹平，精度崩塌。GPTQ/AWQ 的精妙在于引入 Hessian 感知或激活感知来处理非均匀分布，说明量化必须基于数据分布，不能一刀切。

常见误区 2：混淆“训练时量化”和“训练后量化”的适用性。GPTQ/AWQ 是无梯度更新的 PTQ 方法，它们不动原模型权重，只做后处理；若模型本身对量化极敏感（如小模型、或存在大量异常激活通道），仍需 QAT。很多工程师以为 AWQ/GPTQ 可以无条件复现 FP16 精度，实际它们的感知机制只能缓解而无法消除某些结构性脆弱。

思考题：在 GPTQ 的推导中，为何使用的 Hessian 矩阵是 2 X^T X（其中 X 是该层的输入激活）而不是单纯的权重二阶导？请从“量化误差如何通过激活传播到最终 loss”的角度解释，并说明若忽略激活统计而仅用权重分布做优化，会引入什么样的系统性偏差。
