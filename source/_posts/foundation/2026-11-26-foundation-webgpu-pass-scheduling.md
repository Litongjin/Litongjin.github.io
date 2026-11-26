---
title: "每日基础技术总结 · 2026-11-26 · WebGPU 底层概念：Compute Pass 与 Render Pass 的调度模型"
date: 2026-11-26 16:40:06
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-26 · WebGPU 底层概念：Compute Pass 与 Render Pass 的调度模型

## 📚 今日主题

> **WebGPU 底层概念：Compute Pass 与 Render Pass 的调度模型**（前端底层与计算机基础）

### 1. 核心概念速览
Compute Pass 与 Render Pass 是 WebGPU 中两类命令编码执行域。Compute Pass 用于通用并行计算，以 dispatchWorkgroups 描述 workgroup 级调度；Render Pass 用于图形渲染，以 color/depth/stencil attachments、draw 命令与光栅化状态描述帧内渲染目标。它们解决的核心问题不是立即调用 GPU，而是把资源访问、管线状态和执行依赖显式录制为命令缓冲，再提交给驱动和硬件调度。机制上，CPU 侧通过 GPUCommandEncoder 记录 pass，pass.end() 形成执行/资源状态边界，encoder.finish() 生成一次性 GPUCommandBuffer，queue.submit() 将命令交给 GPU queue；驱动负责状态验证、屏障插入、纹理布局转换与硬件引擎分配。它在体系中位于应用逻辑与底层图形/异构计算栈之间：上接 JS/WASM、WGSL 着色器和资源绑定，下接 Vulkan、Metal、D3D12 的命令队列、同步原语和 GPU 命令处理器。专业前端工程师必须掌握它，因为 WebGPU 的 AI kernel、矩阵乘法、图像处理、离屏合成与高性能 Canvas 都依赖 compute pass 调度；而渲染合成、延迟渲染与 tile-based GPU 优化依赖 render pass 的 attachment/load/store 模型。不理解 pass 调度，就无法分析 GPU 内存可见性、命令重叠执行和资源竞争。

### 2. 底层原理剖析
1. 命令录制模型：device.createCommandEncoder() 后，beginComputePass 或 beginRenderPass 只是进入命令录制上下文。setPipeline、setBindGroup、dispatchWorkgroups、draw 都不会立即执行，而是写入命令缓冲。
2. Compute Pass 调度域：计算 pass 没有 attachments 和固定渲染输出，核心调度单位是 workgroup。dispatchWorkgroups(x, y, z) 记录的是启动多少个 workgroup；每个 workgroup 在 GPU 上映射到一组线程/invocation，可被硬件分发到 compute units。多个 compute pass 之间、以及 pass 与 copy/render pass 之间的资源依赖，由 pass 边界和驱动屏障保证可见性。
3. Render Pass 调度域：render pass 在开始时声明 color/depth/stencil attachments、loadOp/storeOp 与 render area，结束时通过 end() 释放/转换纹理状态。draw 命令在该 attachment 域内执行顶点/片元管线。tile-based GPU 上，render pass 还对应 tile memory 的加载与写回，因此 loadOp/storeOp 会直接影响带宽。
4. Queue 与执行：WebGPU 暴露的 GPUQueue 是逻辑提交入口。submit 顺序定义提交序，但 GPU 并非机械逐条同步执行；驱动和硬件会根据资源依赖、命令类型、引擎能力进行重叠、乱序或批处理。没有依赖的命令可并发，有依赖的命令由屏障形成 happens-before。
5. 与前端已有模型对比：WebGL 更像立即状态机，gl.bindFramebuffer、gl.drawElements 直接改变全局状态并触发驱动命令；WebGPU 更像把一帧工作打包成可验证命令流。若对比接口模型：TS 的 interface 是编译期结构约束，运行时会擦除；Java 的 interface 具有运行时类型意义；WebGPU 中 ComputePassEncoder 与 RenderPassEncoder 的差异不是纯类型标记，而是被验证器、驱动和硬件感知的运行时执行域差异。
6. 依赖语义：pass 内部命令可能被重叠或重排，跨 dispatch/draw 的数据依赖不能靠 CPU 调用顺序保证。若后续工作依赖前一计算结果，通常应拆分为不同 pass、不同 submit，或使用底层同步/原子机制；shader 内部 workgroupBarrier 只能同步同一 workgroup，不能跨 workgroup 或跨 dispatch。

### 3. 基础代码与实战验证
```text
// 目标：验证 pass 是命令录制与提交调度，而不是同步执行。
// 以下为 WebGPU API 级伪代码，省略完整 pipeline/shader 创建，聚焦调度模型。
const encoder = device.createCommandEncoder(); // CPU 开始录制命令缓冲，此时 GPU 未执行

const cpass = encoder.beginComputePass(); // 进入 compute 域：无 attachments，只有计算资源绑定与 dispatch
cpass.setPipeline(computePipeline); // 绑定 compute pipeline，决定 shader、布局与本地内存
cpass.setBindGroup(0, computeBindGroup); // 记录资源访问范围，验证器据此检查读写兼容
cpass.dispatchWorkgroups(64, 1, 1); // 记录启动 64 个 workgroup；真正执行发生在 submit 后
cpass.end(); // 结束 compute pass：形成资源状态与可见性边界，便于后续 pass 依赖此输出

const rpass = encoder.beginRenderPass({ // 进入 render 域：必须声明输出 attachments 与 load/store
  colorAttachments: [{
    view: colorView,
    clearValue: { r: 0, g: 0, b: 0, a: 1 },
    loadOp: 'clear',
    storeOp: 'store'
  }]
});
rpass.setPipeline(renderPipeline); // 绑定图形管线：顶点/片元、光栅化、blend 等状态
rpass.setBindGroup(0, renderBindGroup); // 若读取 compute 写出的 buffer/texture，依赖来自 pass 边界
rpass.draw(3, 1, 0, 0); // 记录 draw；光栅化与片元写入不会立即发生
rpass.end(); // 结束 render pass：驱动可据此插入 attachment layout transition

const commandBuffer = encoder.finish(); // 生成一次性命令缓冲，之后不能再次录制
device.queue.submit([commandBuffer]); // 提交到 GPUQueue；驱动解码、插屏障、交由硬件调度
device.queue.onSubmittedWorkDone().then(() => {
  // CPU 侧确认该 submit 已完成；不要假设 submit 返回时 GPU 已写完资源。
});
```

### 4. 常见误区与进阶思考
1. 把 pass 当成立即执行或把 submit 当成同步完成：beginComputePass/dispatchWorkgroups/beginRenderPass/draw 只是录制命令；queue.submit 只是把命令缓冲交给队列，CPU 返回后 GPU 可能尚未执行。若需要读取结果，必须通过 copyBufferToBuffer + mapAsync、onSubmittedWorkDone、或后续 submit 中的资源依赖来保证完成与可见。
2. 误以为同一 compute pass 中的多个 dispatch 会自动串行并产生跨 dispatch 可见性：pass 是调度与资源使用域，硬件可能并发或重排。workgroupBarrier 只同步同一 workgroup，不能跨 workgroup；跨 dispatch 的数据依赖通常需要拆分 compute pass、使用原子/存储屏障语义，或在更高层通过 pass 边界建立同步。进阶思考：如果一个 compute pass 写入 storage texture，后续 render pass 要采样该纹理，且不能 CPU 回读，你应如何组织资源 usage、pass 边界与 bind group，以保证纹理从 storage write 状态安全转换到 shader read 状态？
