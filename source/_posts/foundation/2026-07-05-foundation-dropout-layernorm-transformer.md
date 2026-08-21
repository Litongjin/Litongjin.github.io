---
title: "每日基础技术总结 · 2026-07-05 · Dropout 与 LayerNorm 在 Transformer 中的位置"
date: 2026-07-05 08:00:00
categories: [技术分享]
tags: ["技术分享", "AI 开发基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-07-05 · Dropout 与 LayerNorm 在 Transformer 中的位置

## 📚 今日主题

> **Dropout 与 LayerNorm 在 Transformer 中的位置**（AI 开发基础）

### 1. 核心概念速览
Dropout 与 LayerNorm 是深度神经网络中的两种基础正则化与归一化技术。Dropout 在训练阶段以概率 p 随机将神经元的输出置零，并乘以 1/(1-p) 保持期望不变，本质是通过对大量子网络的隐式集成降低过拟合；推理阶段为恒等映射。LayerNorm 在特征维度（对每个样本独立）计算均值与方差，对输入进行标准化后再进行仿射变换（γ、β），目的是稳定输入分布，缓解深层网络中梯度消失/爆炸，加速收敛。在 Transformer 中，二者通常协同作用于每个子层（多头自注意力或前馈网络）：子层输出经过 Dropout 后与残差连接相加，再进行 LayerNorm（即 Add & Norm 结构）。它们属于模型架构层面的基础组件，直接影响训练稳定性与泛化能力。专业工程师必须理解其数学机理与摆放位置，否则难以针对大规模模型进行训练调优、故障排查及架构改造。

### 2. 底层原理剖析
Dropout 的前向计算：给定输入 x，训练时生成与 x 同形状的掩码 m ~ Bernoulli(1-p)，输出 y = m * x / (1-p)。除以 (1-p) 是为了保持输出期望 E[y]=E[x]。反向传播时，梯度仅流过未被置零的神经元，等价于训练一个共享参数的稀疏网络集成。LayerNorm 的前向计算：对输入 x（形状 [batch, seq_len, d_model]）在最后一维（d_model）上计算均值 μ = mean(x) 和方差 σ² = var(x)，然后归一化：x̂ = (x - μ) / sqrt(σ² + ε)，最后缩放和平移：y = γ * x̂ + β。γ 和 β 是可学习参数，初始为 1 和 0。与 BatchNorm 不同，LayerNorm 不依赖 batch 维统计，因此不受 batch size 影响，适用于变长序列和在线推理。在 Transformer 中，标准 Post-LN 结构为：子层输出 x_sub = SubLayer(x) + Dropout(x_sub_raw)，然后 LayerNorm 在相加后执行，即 x_norm = LayerNorm(x_sub)。Dropout 通常还应用于 embedding 输出和最后的输出层。Pre-LN 则将 LayerNorm 放在子层之前，即 x_norm = LayerNorm(x)，子层输出 = x + Dropout(SubLayer(x_norm))。对比前端已有概念：这类似于前端状态管理中，对组件局部状态做归一化（LayerNorm，不依赖全局）与对全局 store 做归一化（BatchNorm，依赖批统计量）的区别；两者目的都是规范数据分布，但作用域和依赖条件不同，决定了在动态数据流（如序列长度变化、batch 较小）下的稳定性。Dropout 的训练/推理差异则类似于前端工程中 development 与 production 环境下的配置不同——训练时启用随机性以增强泛化，推理时关闭以保持确定性输出。

### 3. 基础代码与实战验证
```text
import numpy as np

def dropout(x, p=0.5, training=True):
    """
    极简 Dropout 前向实现。
    x: 输入数组，任意形状。
    p: 丢弃概率。
    training: 是否训练模式。
    """
    if not training:
        return x  # 推理时恒等映射，保证确定性
    mask = (np.random.rand(*x.shape) > p).astype(np.float32)  # 生成 0/1 掩码，1 表示保留
    return mask * x / (1.0 - p)  # 除以 (1-p) 保持期望不变

def layer_norm(x, gamma, beta, eps=1e-5):
    """
    极简 LayerNorm 前向实现（仅在特征维上归一化）。
    x: 形状 [batch, seq_len, d_model]。
    gamma, beta: 形状 [d_model] 的可学习参数。
    """
    mean = x.mean(axis=-1, keepdims=True)  # 对特征维求均值
    var = ((x - mean) ** 2).mean(axis=-1, keepdims=True)  # 对特征维求方差
    x_hat = (x - mean) / np.sqrt(var + eps)  # 标准化
    return gamma * x_hat + beta  # 仿射变换

# 验证：训练模式时输出期望不变
d = np.ones((2, 4))  # 全 1 输入
out = dropout(d, p=0.5, training=True)
print('Dropout 训练模式输出均值:', out.mean())  # 接近 1.0，证明期望保留
print('Dropout 推理模式输出:', dropout(d, p=0.5, training=False))  # 原样返回

# 验证：LayerNorm 对每个样本独立归一化，与 batch 内其他样本无关
data = np.array([[1.0, 2.0, 3.0, 4.0], [10.0, 20.0, 30.0, 40.0]])
gamma = np.ones(4)
beta = np.zeros(4)
ln_out = layer_norm(data, gamma, beta)
print('LayerNorm 输出均值:', ln_out.mean(axis=-1))  # 接近 0
print('LayerNorm 输出方差:', ln_out.var(axis=-1))  # 接近 1
```

### 4. 常见误区与进阶思考
误区一：在推理阶段仍启用 Dropout。这是对训练/推理行为差异的误解。推理时 Dropout 应完全关闭（恒等映射），否则会引入随机噪声导致输出不稳定。这与前端工程中混淆 development 与 production 环境配置的危害类似，但后果更隐蔽——模型精度忽高忽低。
误区二：将 BatchNorm 直接移植到 Transformer。BatchNorm 依赖 batch 维统计量，在 batch size 很小或序列长度动态变化时，统计量剧烈波动，导致训练不稳定；且推理时需要使用移动平均，增加了复杂度。LayerNorm 对每个样本独立归一化，天然适配序列模型。
进阶思考：在 Pre-LN 和 Post-LN 两种结构中，梯度从输出层向输入层反向传播时，残差连接的路径有何不同？为什么 Pre-LN 在深层 Transformer 中往往更稳定，而 Post-LN 在浅层模型中表现更好？请从梯度范数和恒等映射的角度分析。
