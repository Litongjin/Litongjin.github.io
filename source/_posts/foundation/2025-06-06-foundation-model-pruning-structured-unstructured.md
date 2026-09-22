---
title: "每日基础技术总结 · 2025-06-06 · 模型剪枝：结构化剪枝与非结构化剪枝"
date: 2025-06-06 20:00:00
categories: [技术分享]
tags: ["技术分享", "AI 开发基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2025-06-06 · 模型剪枝：结构化剪枝与非结构化剪枝

## 📚 今日主题

> **模型剪枝：结构化剪枝与非结构化剪枝**（AI 开发基础）

### 1. 核心概念速览
模型剪枝（Model Pruning）是一种通过移除神经网络中冗余参数以压缩模型体积、降低计算复杂度并提升推理速度的技术。其本质是对高维参数空间进行稀疏化或低秩化变换，旨在解决深度学习模型中存在的过参数化（Over-parameterization）问题。

1. 结构化剪枝（Structured Pruning）：沿张量维度（如通道、层、滤波器）进行整块剔除。它直接改变模型的拓扑结构（Architecture），导致底层存储格式和计算图发生变化，能直接映射到硬件加速指令集（如GEMM的变长矩阵乘法），实现真正的部署加速。
2. 非结构化剪枝（Unstructured Pruning）：对单个权重值进行阈值截断，将接近零的值设为精确零。它不改变模型维度，仅产生稀疏矩阵（Sparse Matrix）。虽然显著减少存储空间，但由于数据分布随机，难以利用现有硬件向量单元的高效并行性，需专用稀疏计算内核才能发挥实际效能。

对于全栈/后端工程师，掌握此知识点意味着理解模型从'黑盒预测'到'工程可部署实体'的转化关键路径，以及模型表示（Representation）与执行效率（Execution Efficiency）之间的权衡机制。

### 2. 底层原理剖析
核心差异在于'粒度'与'计算图重构'。

【非结构化剪枝】机制：
1. 重要性评估：遍历所有标量权重 w_ij，根据绝对值大小或其他梯度敏感度指标打分。
2. 阈值裁剪：设定阈值 T，若 |w_ij| < T，则置 w_ij = 0。
3. 结果形态：得到大量离散的 0，形成稀疏矩阵。存储时需采用 CSR/CSC 等压缩格式，而非原生 Dense Tensor。
4. 计算瓶颈：现代 GPU/NPU 基于 SIMD/SIMT 架构，强制对齐内存访问。稀疏模式打破了对齐规律，引发分支预测失败和低效的内存读取，除非使用硬件级的稀疏卷积算子。

【结构化剪枝】机制：
1. 组级评估：按滤波器（Filter）、通道（Channel）或层（Layer）为单位聚合重要性分数（如 L1/L2 norm of filter weights）。
2. 批量剔除：移除重要性低的整个维度。例如，移除第 k 个滤波器的输出通道。
3. 结果形态：矩阵维度缩小（如 Conv2D 输入 64x64x64 -> 64x64x32）。
4. 计算优势：重新定义了 Convolution 算子的输入输出张量形状，可直接复用标准的稠密 GEMM/CuBLAS 内核，无需特殊稀疏逻辑即可在通用硬件上获得线性级别的加速。

对比前端概念：
- 类似 TS 中的 Type Guard vs Interface Implementation。
- 非结构化剪枝像保留接口定义但内部逻辑大量返回 null/undefined，调用方仍需处理空值检查，运行时开销未减；结构化剪枝像是直接删除了某个类的方法签名并调整了依赖注入列表，编译期/链接期即完成瘦身，运行时不再存在该路径。

### 3. 基础代码与实战验证
```text
// Python 伪代码演示两种剪枝对张量维度的影响本质
import torch.nn as nn
import torch.nn.functional as F

def pruned_forward():
    pass

# 假设 input_tensor shape: [Batch, C_in, H, W]
# 原始卷积核 weights shape: [C_out, C_in_group, K_h, K_w]

# 【非结构化剪枝】：操作对象是元素级标量
# 即使剪掉 90% 的参数，weights 的 shape 依然是 [C_out, C_in_group, K_h, K_w]
# 只是大部分值为 0。如果直接用 dense tensor 运算，F.conv2d 依然会逐元素相乘，无效计算量大。
# 需要配合 sparse backend 才能优化。
def unstructured_pruning(weights):
    threshold = 0.01
    # mask = (weights.abs() > threshold).float()
    # pruned_weights = weights * mask
    return weights # shape unchanged

# 【结构化剪枝】：操作对象是 Channel/Filter 级别
# 移除第 i 个滤波器后，C_out 维度直接减小
# 这改变了后续层的输入维度 C_in，迫使网络结构重组
# 这是真正的'剪枝'，因为物理连接数减少了

# 示意：计算每个输出的 L2 norm
filter_norms = torch.sqrt(torch.sum(torch.abs(weights)**2, dim=[1, 2, 3]))
cut_indices = filter_norms.sort().indices[:keep_percentage] 

# 提取保留的滤波器
pruned_weights_structured = weights[cut_indices]

# 关键区别：
# pruned_weights_structured.shape[0] < original_weights.shape[0]
# 这意味着下一层输入的 Channels 数量必须同步缩减，否则维度不匹配报错。
# 这种维度缩减是直接映射到矩阵乘法大小的变化。
```

### 4. 常见误区与进阶思考
1. 误区：认为剪枝等于量化（Quantization）。
   纠正：量化是将浮点数精度从 FP32 降到 INT8/BF16，属于精度压缩，不影响拓扑；剪枝是移除参数，属于稀疏化压缩，影响拓扑（结构化）或数据密度（非结构化）。两者常结合使用，但数学机理完全不同。

2. 误区：认为非结构化剪枝在所有硬件上都能加速。
   纠正：在非专用硬件（如普通 CPU 或标准 GPU）上，非结构化剪枝往往因缓存命中率下降和指令分支开销导致速度反而不如 Dense 模型。其核心价值在于极致压缩体积（Storage Size），而非必然带来推理延迟（Latency）的降低。

思考题：
在移动边缘计算场景下，若目标芯片不支持硬件级稀疏算子加速，且对功耗极其敏感（要求最小化内存带宽压力），你会选择非结构化剪枝还是结构化剪枝？为什么？请从内存访问模式（Memory Access Pattern）的角度论证。
