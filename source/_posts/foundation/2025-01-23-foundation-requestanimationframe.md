---
title: "每日基础技术总结 · 2025-01-23 · requestAnimationFrame 与宏微任务在渲染中的时序"
date: 2025-01-23 20:00:00
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础（扩展）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2025-01-23 · requestAnimationFrame 与宏微任务在渲染中的时序

## 📚 今日主题

> **requestAnimationFrame 与宏微任务在渲染中的时序**（前端底层与计算机基础（扩展））

### 1. 核心概念速览
requestAnimationFrame (rAF) 是浏览器提供的用于同步绘制操作的 API，其核心机制是将回调函数注册到浏览器的每一帧渲染队列中。它解决的核心问题是避免屏幕撕裂、保证视觉流畅度并优化后台 Tab 的性能表现。在时序上，rAF 回调执行于 DOM 更新之后、重排（Reflow）与重绘（Repaint）之前。对于资深工程师而言，掌握此概念的本质在于理解‘UI 线程的周期性调度’与‘JS 主线程执行栈’之间的交互契约，这是构建高性能动画、复杂图表及 Web Worker 通信的基础，也是理解浏览器内核如何将逻辑执行转化为像素输出的关键节点。

### 2. 底层原理剖析
浏览器渲染管线遵循固定周期（通常对应显示器刷新率，如 60Hz/16.67ms）。每个帧的处理流程严格如下：
1. Main Thread: 执行当前 Task Queue 中的宏任务（Macrotask）。
2. Microtask Checkpoint: 清空微任务队列（Microtask Queue，包括 Promise.then, MutationObserver）。
3. rAF Callbacks: 遍历并执行所有注册的 requestAnimationFrame 回调。
4. Style Calculation: 计算各元素的样式（Styles），生成样式表。
5. Layout (Reflow): 计算几何布局，确定元素大小和位置。
6. Paint: 生成绘制指令列表。
7. Composite: 将图层合并为位图发送给 GPU。

对比 TS/Java 接口：TS 接口是静态编译时的类型约束，仅存在于源码阶段，编译后消失；而 rAF 是运行时的动态调度钩子，依赖 V8 引擎与浏览器 GUI 线程的内核级协作。rAF 并非 JavaScript 原生定时器（如 setTimeout/setInterval），后者无法感知屏幕刷新周期，导致掉帧或重复触发。rAF 强制 JS 执行暂停以等待 UI 更新完成前的特定窗口期，体现了‘事件循环（Event Loop）’对硬件驱动行为的适配。

### 3. 基础代码与实战验证
```text
// 验证 rAF 与宏微任务的相对时序
// 关键在于观察日志输出顺序，而非依赖时间戳

console.log('Start Macro Task'); // 当前宏任务开始

Promise.resolve().then(() => {
    console.log('Micro Task 1'); // 第一个微任务
});

setTimeout(() => {
    console.log('Next Macro Task (setTimeout)'); // 下一个宏任务
}, 0);

requestAnimationFrame((timestamp) => {
    console.log('rAF Callback', timestamp); // rAF 回调
});

console.log('End of Current Macro Task');

// 预期输出逻辑解释：
// 1. 'Start...' 和 'End...' 在当前宏任务内同步执行。
// 2. Micro Task 立即被推入微任务队列，并在当前宏任务结束后、rAF 前执行。
// 3. rAF 回调不会在当前宏任务中执行，也不会像 setTimeout 那样进入下一个宏任务轮次，
//    而是插入到 UI 渲染管线的特定阶段（DOM 更新后，布局计算前）。
// 4. 若 DOM 操作发生在 rAF 内部，它将立即生效并影响后续的 Layout/Paint 阶段。
// 5. setTimeout 的回调将推迟到下一帧的宏任务阶段执行。
```

### 4. 常见误区与进阶思考
['误区一：认为 rAF 等同于 setTimeout(fn, 16)。实际上，rAF 的执行时机取决于浏览器是否准备好下一帧，且必须在当前宏任务结束后、绘制调用前触发。若在主线程进行耗时极长的同步计算阻塞了主线程，后续的所有宏任务和 rAF 均会被延迟，导致严重卡顿（Jank），而 setTimeout 至少能按时间线排队，但依然受限于主线程空闲状态。', '误区二：误以为在 rAF 中修改 DOM 样式一定会导致强制同步布局（Forced Synchronous Layout）。虽然在 rAF 中读取 layout 属性（如 offsetHeight）会触发强制布局，但写入 style 属性通常只是积累绘图指令，直到浏览器决定真正执行 Paint。因此，在 rAF 中进行批量 DOM 写入并仅在最后读取布局属性以触发一次性重排，是性能最优解。', '思考题：在 Service Worker 或 Web Worker 等非 UI 线程中能否直接使用 requestAnimationFrame？如果不能，请从 OS 进程隔离、IPC（进程间通信）开销以及浏览器主线程对 GUI 资源独占权的角度，设计一种跨线程通知 UI 线程发起 rAF 调度的最小化方案，并分析其潜在的性能瓶颈。']
