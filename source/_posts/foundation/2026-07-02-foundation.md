---
title: "每日基础技术总结 · 2026-07-02 · 模型量化：对称/非对称量化与零点"
date: 2026-07-02 08:00:00
categories: [技术分享]
tags: ["技术分享", "AI 开发基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-02 · 模型量化：对称/非对称量化与零点

## 📚 今日主题

> **模型量化：对称/非对称量化与零点**（AI 开发基础）

### 1. 核心概念速览
模型量化是将浮点权重和激活从 FP32 映射到低比特整数（如 INT8）的线性仿射变换。其本质是用离散整数网格近似连续实数，通过缩放因子（scale）和零点（zero_point）唯一确定映射关系。它解决模型体积过大、内存带宽不足、浮点运算慢的问题，使推理可运行在仅支持整数指令集的硬件上。在 AI 体系中，量化属于模型压缩与推理优化层，与剪枝、蒸馏并列。专业工程师必须掌握，因为量化精度损失与超参选择直接相关，且 TensorRT、XNNPACK 等加速库的底层均依赖这一映射机制。

### 2. 底层原理剖析
量化映射是仿射变换：r = s(q - z)，其中 r 为浮点值，q 为量化整数，s 为缩放因子（浮点），z 为零点（整数）。反量化即由 q 求解 r。给定浮点范围 [min, max] 和量化范围 [q_min, q_max]，有：

s = (max - min) / (q_max - q_min)
z = q_min - round(min / s)

量化时：
q = clip(round(r / s) + z, q_min, q_max)

对称量化强制 z = 0，且量化范围关于 0 对称（如 [-127, 127]），因此 s = max(|min|, |max|) / 127。非对称量化不强制 z = 0，可适应任意浮点分布，典型为 [0, 255] 无符号。误差来源有裁剪（超出范围被截断）和舍入（取整导致离散化）。

与前端知识体系的对比：Canvas 的 ImageData 用 Uint8ClampedArray 存储 0-255 的像素值，底层即是一种非对称量化（浮点颜色 0-1 映射到整数 0-255，零点为 0 但范围不对称）；而 TypeScript 的类型约束是编译期行为，不改变运行时数值表示，量化则是在运行时真实改变数值存储与计算。

### 3. 基础代码与实战验证
```text
import numpy as np

def quant_asym(x, bits=8):
    # 非对称量化：浮点范围 [min, max] 映射到无符号整数 [0, 2^bits - 1]
    q_min, q_max = 0, (1 << bits) - 1
    x_min, x_max = x.min(), x.max()
    if x_min == x_max:
        scale = 1.0
        zero_point = q_min - int(round(x_min / scale))
    else:
        # 计算缩放因子：浮点范围宽度 / 量化范围宽度
        scale = (x_max - x_min) / (q_max - q_min)
        # 零点：量化整数 0 对应的浮点值对应的整数，用 round 消除偏移
        zero_point = q_min - int(np.round(x_min / scale))
    # 量化：先除以 scale，加零点，再裁剪到有效范围
    q = np.clip(np.round(x / scale) + zero_point, q_min, q_max)
    # 反量化：还原浮点值（存在误差）
    deq = scale * (q - zero_point)
    return q.astype(np.uint8), scale, zero_point, deq

def quant_sym(x, bits=8):
    # 对称量化：零点固定为 0，量化范围 [-2^(bits-1), 2^(bits-1)-1]
    q_min, q_max = -(1 << (bits - 1)), (1 << (bits - 1)) - 1
    max_abs = np.max(np.abs(x))
    if max_abs == 0:
        scale = 1.0
    else:
        # 用 127 作为分母，保证正负对称且避免溢出（-128 留作余量）
        scale = max_abs / ((1 << (bits - 1)) - 1)
    # 量化：直接除以 scale，零点为 0，无需加偏移
    q = np.clip(np.round(x / scale), q_min, q_max)
    deq = scale * q
    return q.astype(np.int8), scale, 0, deq

# 验证：对一个非对称分布的张量分别做两种量化
x = np.array([0.1, 0.5, 1.0, -0.2], dtype=np.float32)
q_a, s_a, z_a, deq_a = quant_asym(x)
q_s, s_s, z_s, deq_s = quant_sym(x)
print('原始:', x)
print('非对称量化:', q_a, 'scale:', s_a, 'zero_point:', z_a, '反量化:', deq_a)
print('对称量化:', q_s, 'scale:', s_s, 'zero_point:', z_s, '反量化:', deq_s)
```

### 4. 常见误区与进阶思考
误区 1：认为量化只是简单的四舍五入，忽略 scale 和 zero_point 的仿射映射。这会导致反量化公式错误，甚至把零点偏移当成噪声，无法正确恢复浮点值。

误区 2：认为对称量化一定比非对称量化精度差。实际上，对称量化对接近 0 对称的分布（如卷积权重）效率高，非对称量化对偏置分布（如 ReLU 激活值全为正）更优，但引入 zero_point 会带来额外计算开销。

思考题：若张量所有值均为正（如 [0.1, 0.5, 1.0]），使用 8 位对称量化（INT8）与非对称量化（UINT8）相比，量化误差有何本质区别？试计算两者的 scale 和有效精度，并解释为什么对称量化会浪费一半量化范围。
