---
title: "每日基础技术总结 · 2024-12-11 · 训练后量化 PTQ 与量化感知训练 QAT"
date: 2024-12-11 20:00:00
categories: [技术分享]
tags: ["技术分享", "AI 开发基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-12-11 · 训练后量化 PTQ 与量化感知训练 QAT

## 📚 今日主题

> **训练后量化 PTQ 与量化感知训练 QAT**（AI 开发基础）

### 1. 核心概念速览
量化是降低模型精度以换取推理效率与存储节省的技术，核心是将浮点32（FP32）映射至低位宽格式（如INT8）。训练后量化（PTQ, Post-Training Quantization）：在完整模型训练收敛后，对冻结的参数和激活值进行离线校准，计算统计分布并应用缩放因子（Scale）与零点（Zero-point），不改变优化目标，速度极快但可能损失精度。

量化感知训练（QAT, Quantization-Aware Training）：在训练前向传播中模拟量化噪声（添加假性量化节点FakeQuantize），反向传播时通过直通估计器STE（Straight-Through Estimator）让梯度绕过量化截断，使模型主动学习适应低位宽的权重分布。本质上是将对量化的敏感性内化为训练过程的一部分，显著优于PTQ，尤其适用于Transformer等敏感架构。

### 2. 底层原理剖析
机制对比：
1. PTQ（静态映射）：类似编译期的常量折叠或类型强转。扫描验证集统计Activation的最大绝对值（Max Abs），动态计算 Scale = Max / 127，Zero Point 对齐整数域。推理时执行 INT8 MatMul + INT32 Accumulate + Dequantize（乘回Scale）。无梯度流干扰。
2. QAT（动态模拟）：类似运行时的异常捕获与补偿。在前向路径插入 FakeQuantize Ops，强制激活值在 [0, 255] 或 [-128, 127] 之间截断取整。反向传播时，Math.abs(grad) 穿过 FakeQuant 节点直接传递（STE原理：dL/dW ≈ dL/d(W_float)），让参数在‘受约束’的loss表面优化。

前端类比差异：JS的动态类型（Float）vs TS的严格静态类型（Int/Uint8）。PTQ如同构建后直接将 JS Bundle 替换为 Wasm (WebAssembly)，若未做边界测试可能导致运行时逻辑错误；QAT 如同在 TS 编译阶段开启 `--strictNullChecks` 和 `--noImplicitAny`，虽然配置繁琐（需加FakeOp），但在编译期（训练期）就解决了类型溢出的隐患。

### 3. 基础代码与实战验证
```text
// PyTorch QAT 底层原理极简复现（基于 nn.Module 手动模拟 FakeQuantize）
import torch
import torch.nn as nn

class ManualFakeQuant(nn.Module):
    def __init__(self, num_bits=8, symmetric=True):
        super().__init__()
        self.num_bits = num_bits
        self.quant_min = -(2**(num_bits-1)) if symmetric else 0
        self.quant_max = 2**(num_bits-1) - 1 if symmetric else 2**num_bits - 1
        
    def forward(self, x):
        # 1. 计算 Scale & ZeroPoint (实际工程中通常用观察值 Observation)
        scale = (x.max() - x.min()) / (self.quant_max - self.quant_min)
        zero_point = -x.min() / scale if not symmetric else 0.0
        
        # 2. 量化：归一化到 [0,1] -> 映射到整数域 -> 截断取整 -> 反归一化
        q_x = x / scale + zero_point
        q_x_clipped = torch.clamp(q_x, self.quant_min, self.quant_max)
        x_dequant = (q_x_clipped - zero_point) * scale
        
        # 3. 关键 STE (Straight-Through Estimator): 
        # loss.backward() 时，梯度必须像没发生截断一样，原样传递给 x
        return x_dequant.detach() - x + x

# 调用示意
model = YourModel()
fake_quant_module = ManualFakeQuant(num_bits=8)
for param in model.parameters():
    param.requires_grad = True # 允许微调

optimizer = torch.optim.Adam(model.parameters(), lr=1e-4)
# 在训练中插入 fake_quant_module.forward(input_tensor) 即可实现 QAT
```

### 4. 常见误区与进阶思考
误区1：认为 QAT 只是简单的‘剪枝后再量化’。QAT 需要额外的几个 Epoch 进行微调和 Fine-tuning，因为初始的随机初始化权重通常不具备抵抗低位宽扰动的鲁棒性，直接使用预训练权重进入 QAT 会导致 Loss 剧烈震荡。

误区2：混淆对称与非对称量化策略的选择时机。PTQ 常用非对称以处理 Relu 后的单侧偏置；而硬件加速器（如 NPU/TensorRT）往往偏好对称量化以简化 INT8 乘加单元的实现，强行使用非对称可能导致内核加速失效。

思考题：在 Transformer 模型中，为什么 Multi-Head Attention 的 Softmax 层通常不适合进行低比特量化？请从数值稳定性与 softmax 的数学特性角度分析其对量化噪声的敏感度。
