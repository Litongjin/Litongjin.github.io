---
title: "每日基础技术总结 · 2026-07-04 · 混合精度训练：FP16 的梯度下溢与损失缩放"
date: 2026-07-04 08:00:00
categories: [技术分享]
tags: ["技术分享", "AI 开发基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-04 · 混合精度训练：FP16 的梯度下溢与损失缩放

## 📚 今日主题

> **混合精度训练：FP16 的梯度下溢与损失缩放**（AI 开发基础）

### 1. 核心概念速览
混合精度训练是指在训练过程中同时使用 FP32 与 FP16 两种精度存储与计算，核心动机是利用现代 GPU（如 Tensor Core）对 FP16 矩阵乘法的加速能力，同时通过损失缩放（Loss Scaling）对抗 FP16 有限动态范围导致的梯度下溢。本质问题是：FP16 的指数位仅 5 位，最小正规格化数为 2^-14 ≈ 6.1e-5，非规格化数可下探至 2^-24 ≈ 5.96e-8，而反向传播中梯度常远小于此量级，直接以 FP16 存储会截断为 0，导致权重无法更新。损失缩放通过在反向传播前将损失乘以一个大因子（如 1024），使梯度整体上移进入 FP16 可表示范围，更新权重前再除以该因子，从而在保持 FP16 计算加速的同时逼近 FP32 的数值稳定性。它位于 AI 基础设施层的数值计算与硬件指令集交汇处，是训练大模型（如 GPT、BERT）的默认策略之一。专业工程师必须掌握它，因为模型规模越大，梯度分布越稀疏，下溢概率越高；且混合精度已内置于 PyTorch AMP、NVIDIA Apex 等库，不理解其底层机制将无法诊断训练发散、精度异常或性能回退问题。

### 2. 底层原理剖析
底层机制分四步：
1. 前向传播：权重保持 FP32 主副本，每层计算时将其转为 FP16 参与矩阵乘；中间激活值以 FP16 存储，必要时保存 FP32 副本用于梯度计算（取决于实现策略）。
2. 损失缩放：前向损失 L 乘以缩放因子 S（通常 1024），得到 L' = L * S。该因子需足够大，使反向传播中最大梯度不超过 FP16 最大值 65504，同时最小梯度不小于 2^-24。
3. 反向传播：从 L' 开始求梯度，所有梯度以 FP16 计算与存储。由于链式法则，每一层梯度都乘以 S，相当于梯度值整体放大，避免下溢。
4. 权重更新：将 FP16 梯度转为 FP32，除以 S 得到真实梯度，再更新 FP32 权重主副本。若发现梯度溢出（inf/NaN），则跳过该步，将 S 减半并重新前向/反向。
与前端概念的对比：这类似于 TypeScript 的 `number` 与 `bigint` 的精度取舍——TS 中 `number` 是 IEEE 754 双精度，`bigint` 是任意精度，但 `bigint` 运算慢；混合精度不是简单“用低精度代替高精度”，而是“高精度存储，低精度计算，通过缩放桥接动态范围差异”。更贴切的类比是 HTTP/2 与 HTTP/1.1 的多路复用：前者将多个请求交错在一个连接上，必须解决队头阻塞问题（对应梯度下溢）；损失缩放就像为每个请求分配独立流 ID，让小请求（小梯度）不被大请求（大梯度）淹没。本质上都是“在受限的共享资源中保证小信号的生存”。

### 3. 基础代码与实战验证
以下用纯 PyTorch 实现最小混合精度训练循环，不依赖 `torch.cuda.amp`，展示损失缩放原理：

```python
import torch

# 模型：单线性层，权重初始化为小值，便于产生小梯度
model = torch.nn.Linear(8, 1).double()  # 内部保持 FP32/FP64 主副本
model.half()  # 转换为 FP16 权重（真实场景会保留 FP32 主副本，这里简化）

optimizer = torch.optim.SGD(model.parameters(), lr=0.01)
loss_fn = torch.nn.MSELoss()

# 输入与目标，使用 FP16
x = torch.randn(4, 8, dtype=torch.float16)
y = torch.randn(4, 1, dtype=torch.float16)

# 损失缩放因子，初始化为 1024
scaler = 1024.0

for step in range(3):
    optimizer.zero_grad()

    # 前向：FP16 计算，计算 FP32 损失（防止损失自身下溢）
    pred = model(x)          # FP16 预测
    loss = loss_fn(pred.float(), y.float())  # 转为 FP32 计算损失

    # 损失缩放：将损失乘以 scaler，梯度将按相同比例放大
    scaled_loss = loss * scaler

    # 反向传播：梯度以 FP16 存储（实际 autograd 会用 FP16），此时梯度被放大
    scaled_loss.backward()

    # 检查梯度是否溢出（inf/NaN），若溢出则跳过更新并降低 scaler
    grad_ok = all(torch.isfinite(p.grad).all() for p in model.parameters())
    if not grad_ok:
        scaler /= 2.0
        print(f"Step {step}: overflow, scaler halved to {scaler}")
        continue

    # 更新权重前，将梯度除以 scaler 还原，并转为 FP32 更新
    for p in model.parameters():
        p.grad.data = p.grad.float() / scaler
    optimizer.step()

    print(f"Step {step}: loss={loss.item():.6f}, grad_scale={scaler}")
```

关键行注释：
- `scaled_loss = loss * scaler`：将损失放大，后续梯度自动乘以 scaler，使小梯度进入 FP16 可表示范围。
- `torch.isfinite(p.grad)`：检测 FP16 梯度是否溢出，若溢出说明 scaler 过大，需减半重试（真实实现会跳过本轮更新）。
- `p.grad.float() / scaler`：还原真实梯度，再执行 FP32 权重更新，保证权重精度不被破坏。
该代码可直接运行，观察 loss 是否正常下降；若去掉缩放，梯度极小（线性层输出接近 0）时梯度会下溢为 0，loss 不下降。

### 4. 常见误区与进阶思考
误区一：认为混合精度只是“把模型改成 FP16”。实际上必须保留 FP32 权重主副本，否则权重更新时的微小修正量（通常 < 2^-14）会被 FP16 舍入吞掉，训练无法收敛。这与前端开发中用 `Number` 存储时间戳却丢失毫秒精度类似，根因都是“计算精度”与“存储精度”分离。
误区二：认为损失缩放因子越大越好。若 scaler 过大，梯度会溢出为 inf；过小则下溢。正确做法是动态调整：初始较大，检测到 inf 后减半，并在连续若干步无溢出时适当增大（如 NVIDIA 的 `Dynamic Loss Scaling`）。
进阶思考题：当使用 Tensor Core 时，FP16 矩阵乘法的输入必须满足维度为 8 的倍数。假设你的模型隐藏层维度为 10，混合精度训练时为何实际计算会被填充到 16？填充部分如何影响梯度计算？请从内存布局与 Tensor Core 指令集的角度解释，并说明损失缩放是否仍适用于填充区域。
