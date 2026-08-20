---
title: "每日基础技术总结 · 2026-07-30 · 微任务检查点（Microtask Checkpoint）的触发时机"
date: 2026-07-30 08:00:00
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-30 · 微任务检查点（Microtask Checkpoint）的触发时机

## 📚 今日主题

> **微任务检查点（Microtask Checkpoint）的触发时机**（前端底层与计算机基础）

### 1. 核心概念速览
微任务检查点（Microtask Checkpoint）是事件循环中的一个同步例程，由宿主环境（浏览器或 Node.js）在 JavaScript 执行上下文栈清空且微任务队列非空时调用。它按 FIFO 顺序清空整个微任务队列，直到队列为空，并在此过程中同步执行新入队的微任务。其本质是：让 Promise 回调、queueMicrotask、MutationObserver 等微任务在当前同步代码之后、下一个宏任务或渲染之前执行，从而保证异步操作的顺序确定性和低延迟。
它在计算机体系中的位置：属于事件循环（Event Loop）的任务调度机制，是并发模型中实现优先级调度的一环。
专业工程师必须掌握它，因为异步时序分析、性能优化、避免微任务死循环，以及理解 async/await 的底层执行模型都直接依赖于此。

### 2. 底层原理剖析
1. 数据结构与标志位
每个 JavaScript agent（如浏览器标签页或 Node.js 进程）维护一个微任务队列，以及一个布尔标志 performingMicrotaskCheckpoint，用于防止检查点重入。JavaScript 引擎（如 V8）只负责提供微任务队列的入队和清空原语，宿主环境决定何时触发检查点。

2. 触发条件
触发微任务检查点的核心条件是：当前 JavaScript 执行上下文栈（call stack）变为空。具体场景包括：
- 一个宏任务（task）的回调执行完毕后返回宿主环境；
- 一个脚本块（如 <script>）执行完毕；
- 在浏览器渲染更新之前（确保 DOM 变化后的微任务在绘制前完成）；
- 在 Node.js 的事件循环阶段切换时（每个阶段结束后都会执行微任务检查点）。
注意：如果调用栈不为空（例如普通函数嵌套调用），不会触发检查点。

3. 检查点执行过程
宿主调用 performMicrotaskCheckpoint() 时，先检查 performingMicrotaskCheckpoint 标志，若已为 true 则直接返回（避免重入）。然后置为 true，循环执行微任务队列中的任务。执行一个微任务时，它可能向队列尾部添加新微任务，这些新任务会在同一轮检查点中继续执行，直到队列为空。最后置回 false。

伪代码：
function performMicrotaskCheckpoint() {
  if (performingMicrotaskCheckpoint) return;
  performingMicrotaskCheckpoint = true;
  while (!microtaskQueue.isEmpty()) {
    microtaskQueue.dequeue().run();
  }
  performingMicrotaskCheckpoint = false;
}

4. 与事件循环的关系
事件循环从任务队列中取出一个宏任务并执行，执行完成后立即执行微任务检查点。这意味着微任务不会跨宏任务，但可以在同一个宏任务内部多次触发（例如 async 函数中的每个 await 点都会使当前调用栈返回，从而在栈空时触发检查点）。

5. 与前端已有概念的对比
- 与 Node.js 的 process.nextTick 对比：两者都在当前操作之后立即执行，但 nextTick 拥有独立队列，且优先级高于微任务；每次操作（如 I/O 回调）结束后，Node 会先清空 nextTick 队列，再执行微任务检查点。
- 与宏任务对比：宏任务是事件循环调度的最小单元，两个宏任务之间可能插入渲染；微任务检查点则可能在一个宏任务结束后多次触发，只要调用栈清空。因此宏任务可以视为“每任务一次”，微任务检查点是“每调用栈空一次”。

### 3. 基础代码与实战验证
```text
// 验证微任务检查点的触发时机：同步代码后、宏任务内、微任务链
// 运行环境：浏览器控制台或 Node.js

console.log('脚本开始');

setTimeout(() => {
  console.log('宏任务 1 开始');
  // 在宏任务内注册一个微任务
  queueMicrotask(() => console.log('微任务 A'));
  console.log('宏任务 1 结束');
  // 宏任务回调返回，调用栈清空，触发微任务检查点
}, 0);

Promise.resolve().then(() => {
  console.log('微任务 B');
  // 在微任务执行期间再添加一个微任务
  // 检查点会持续清空队列，因此 C 会在同一次检查点中执行
  queueMicrotask(() => console.log('微任务 C'));
});

// 同步代码执行完毕，调用栈清空，第一次微任务检查点触发
console.log('脚本结束');

// 预期输出：
// 脚本开始
// 脚本结束
// 微任务 B
// 微任务 C
// 宏任务 1 开始
// 宏任务 1 结束
// 微任务 A
```

### 4. 常见误区与进阶思考
误区一：认为微任务检查点只在“每个宏任务结束后”或“每轮事件循环末尾”触发。
实际上，触发时机是 JavaScript 调用栈清空时，而不是事件循环轮次。例如在一个宏任务内部，`await Promise.resolve()` 会使当前 async 函数返回，调用栈清空，立即触发微任务检查点，因此 `await` 之后的代码会在同一个宏任务后续同步代码之后、下一个宏任务之前执行。更准确地说，每次宿主从脚本或回调返回时，都会检查栈是否为空并执行检查点。

误区二：认为微任务检查点只执行“当前已存在”的微任务，新入队的微任务要等下一次检查点。
实际上，检查点会循环执行微任务队列直到队列为空，因此一个微任务中若不断添加新微任务（如无限递归 `Promise.resolve().then(...)`），会阻塞事件循环，导致页面卡死。这是微任务死循环的根源。

思考题：
考虑以下代码：
setTimeout(() => {
  Promise.resolve().then(() => console.log('micro'));
  console.log('macro end');
}, 0);
为什么输出顺序是 macro end 然后 micro？如果将 `console.log('macro end')` 替换为 `await Promise.resolve(); console.log('macro end')`，顺序又会如何？请从调用栈清空与微任务检查点触发时机的角度解释。
