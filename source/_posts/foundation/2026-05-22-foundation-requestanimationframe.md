---
title: "每日基础技术总结 · 2026-05-22 · 事件循环的渲染步骤与 requestAnimationFrame 时机"
date: 2026-05-22 08:00:00
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-05-22 · 事件循环的渲染步骤与 requestAnimationFrame 时机

## 📚 今日主题

> **事件循环的渲染步骤与 requestAnimationFrame 时机**（前端底层与计算机基础）

### 1. 核心概念速览
事件循环的渲染步骤（update the rendering）是浏览器在每帧中，在完成任务和微任务后，检查是否需要进行渲染的环节。requestAnimationFrame（rAF）回调在渲染步骤之前同步执行，确保 DOM 变更和样式计算在同一帧内完成，避免画面撕裂。该机制解决的是 JS 异步任务与渲染管线的同步问题。本质上，rAF 是浏览器渲染调度器提供给 JS 的帧级钩子，而非定时器。在整个计算机体系中，它位于浏览器渲染引擎与 JS 运行时之间，是前端异步模型和渲染性能的基石。专业工程师必须掌握，因为它决定了动画、交互和可感知性能的底层行为。

### 2. 底层原理剖析
浏览器事件循环是一个无限循环，每次迭代处理一个 task（宏任务）。其关键流程为：
1. 从 task 队列中取出最早的任务并执行。
2. 执行完该任务后，执行所有可用的 microtask（微任务），直至队列清空。
3. 根据渲染机会（rendering opportunity，通常由帧率、屏幕刷新率、渲染阻塞条件决定）判断是否进入渲染步骤。若进入，则执行以下子步骤：
   a. 触发 resize、scroll 等事件。
   b. 执行 requestAnimationFrame 回调（所有在渲染前注册的回调）。
   c. 执行样式计算（Recalc Style）、布局（Layout）、绘制（Paint）等渲染管线。
rAF 回调的执行时机就在渲染步骤的开头，此时 DOM 变更尚未反映到屏幕上，布局信息也未更新。因此，在 rAF 中修改 DOM 后，浏览器会在同一帧内完成样式和布局计算，然后渲染。
对比前端已有概念：Node.js 事件循环没有渲染步骤，rAF 是浏览器特有的；与 setInterval 不同，rAF 由帧调度驱动，且当标签页后台时自动暂停。与 Vue/React 的 nextTick 不同，nextTick 基于微任务，而 rAF 基于帧，前者在渲染前执行，后者也在渲染前执行，但优先级和触发条件不同。

### 3. 基础代码与实战验证
```text
// 极简验证：观察 rAF 回调的执行时机与事件循环各阶段的关系
const log = console.log;

log('script start');
setTimeout(() => log('macrotask: timeout'), 0);
Promise.resolve().then(() => log('microtask: promise'));
requestAnimationFrame(() => log('rAF callback'));
log('script end');

// 在浏览器中，同步代码执行完当前宏任务后，先清空微任务，因此 'microtask: promise' 一定在 'script end' 之后打印。
// 随后浏览器检查渲染机会。若本轮帧需要渲染，则执行所有 rAF 回调，因此 'rAF callback' 会在渲染步骤前打印。
// 'macrotask: timeout' 是下一个宏任务，通常在该帧渲染完成后或未渲染时作为新任务迭代时执行，因此打印顺序通常为：
// script start -> script end -> microtask: promise -> rAF callback -> macrotask: timeout。

// 验证 rAF 与屏幕刷新率同步（后台标签页暂停）：
let frameCount = 0;
function loop() {
  frameCount++;
  requestAnimationFrame(loop);
}
requestAnimationFrame(loop);
setInterval(() => console.log('fps:', frameCount), 1000);
// 在 60Hz 屏幕下，fps 约为 60；若切换到后台，frameCount 不再增加，证明 rAF 被暂停。
```

### 4. 常见误区与进阶思考
1. 误区：将 rAF 视为定时器，认为它按固定间隔执行。实际 rAF 的触发由显示器的刷新率决定（如 60Hz/120Hz），且在页面不可见或后台时暂停。若需要时间间隔，应使用 timestamp 参数计算增量，而非假设 16.6ms。
2. 误区：认为微任务在渲染之后执行。实际上微任务在当前宏任务结束后、渲染步骤之前执行。这意味着在微任务中大量执行耗时操作会阻塞渲染，导致页面卡顿。

思考题：假设在 rAF 回调中修改了一个元素的 style，然后立即通过 getBoundingClientRect() 读取该元素的布局信息，请问这一行为会触发什么？为什么？这说明了 rAF 与渲染步骤的什么关系？
