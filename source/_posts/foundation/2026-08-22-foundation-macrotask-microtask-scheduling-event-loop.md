---
title: "每日基础技术总结 · 2026-08-22 · 事件循环中的宏任务与微任务调度"
date: 2026-08-22 08:00:00
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-22 · 事件循环中的宏任务与微任务调度

## 📚 今日主题

> **事件循环中的宏任务与微任务调度**（前端底层与计算机基础）

### 1. 核心概念速览
事件循环（Event Loop）是 JavaScript 运行时（浏览器与 Node.js）的底层调度模型，负责协调主线程上的同步代码、宏任务（Macro Task）与微任务（Micro Task）的执行顺序。宏任务包括整个 script 代码块、setTimeout、setInterval、I/O 事件、UI 渲染等；微任务包括 Promise 回调、queueMicrotask、MutationObserver 以及 Node.js 中的 process.nextTick（其优先级高于普通微任务）。本质是优先级分层调度：每个宏任务执行完毕后，运行时会立即清空整个微任务队列，然后才进入下一个宏任务。它解决了异步任务执行顺序的确定性问题，保证所有微任务在当前宏任务与下一个宏任务之间全部完成。该机制是 JavaScript 并发模型的核心，处于运行时底层，是理解异步行为、渲染时机、性能瓶颈（如微任务死循环导致页面卡死）的基石。专业工程师必须掌握，因为异步代码的输出顺序、错误处理、以及基于事件循环的性能优化都建立在对宏任务与微任务调度规则的精确理解之上，而不能停留在“Promise 比 setTimeout 快”的感性认识。

### 2. 底层原理剖析
事件循环底层可抽象为一个主线程 + 两个任务队列（宏任务队列、微任务队列）。核心调度算法如下：

1. 执行一个宏任务（初始为整个 script）。同步代码在该宏任务内按顺序执行。
2. 执行完毕后，检查微任务队列：依次执行所有微任务；如果微任务执行中又产生新的微任务，则继续追加到微任务队列末尾并一并执行，直到微任务队列为空。注意：这是“清空队列”而非“执行一个”。
3. 若需要渲染（浏览器环境），进行渲染。
4. 从宏任务队列中取出下一个宏任务执行。重复上述步骤。

用精确伪代码描述：

```
while (true) {
  MacroTask = queue.macrotask.shift();  // 取一个宏任务
  execute(MacroTask);                   // 同步执行其内部代码
  while (queue.microtask.length > 0) {
    MicroTask = queue.microtask.shift();
    execute(MicroTask);                 // 执行微任务，期间可能新增微任务
  }
  render();                             // 浏览器渲染阶段（非每次循环都发生）
}
```

关键差异在于：宏任务每次只取一个，微任务每次清空全部。因此微任务具有更高的优先级，能保证在当前宏任务结束后、下一个宏任务开始前完成所有状态更新。

与前端已有概念的对比：可类比“宏任务”与“微任务”的关系类似于异步 API 中 `setTimeout` 与 `Promise` 的调度差异——`setTimeout` 把回调放入宏任务队列，`Promise.then` 把回调放入微任务队列；二者都在主线程之外排队，但微任务队列优先被清空。更进一步，浏览器与 Node.js 的事件循环实现也有差异：Node.js 的 `process.nextTick` 队列优先于 Promise 队列；浏览器的 `requestAnimationFrame` 不归属于宏任务/微任务，而是在渲染阶段前执行。这些差异的根源在于底层宿主环境对任务队列的细分，但宏/微任务分层模型是两者共有的骨架。

### 3. 基础代码与实战验证
验证宏任务与微任务调度的最简代码（浏览器或 Node.js 环境均适用）：

```javascript
console.log('script start');

setTimeout(() => {
  console.log('setTimeout');
}, 0);

Promise.resolve().then(() => {
  console.log('promise1');
});

Promise.resolve().then(() => {
  console.log('promise2');
  setTimeout(() => {
    console.log('setTimeout2');
  }, 0);
});

console.log('script end');
```

预期输出顺序：

```
script start
script end
promise1
promise2
setTimeout
setTimeout2
```

逐行注释底层运作：

- `console.log('script start')`：同步执行，属于当前宏任务（script）的一部分。
- `setTimeout(..., 0)`：将回调注册到宏任务队列，延迟计时结束（实际至少 0ms）后等待被取出。
- `Promise.resolve().then(...)`：将回调注册到微任务队列。此时宏任务仍未结束，微任务不会执行。
- 第二个 `Promise.resolve().then(...)`：同样注册到微任务队列，其中内部又注册了一个 `setTimeout`，该回调会进入宏任务队列。
- `console.log('script end')`：同步代码执行完毕，当前宏任务结束。
- 事件循环此时检查微任务队列，按注册顺序依次执行 `promise1`、`promise2`。执行 `promise2` 时，内部 `setTimeout` 被加入宏任务队列。微任务队列清空后，事件循环取出宏任务队列中的 `setTimeout` 回调执行，打印 `setTimeout`。
- 注意：两个 `setTimeout` 虽然都是 0 延迟，但第二个是在微任务执行时才注册的，所以它排在宏任务队列末尾，必须等第一个 `setTimeout` 执行完才会被执行，因此输出 `setTimeout` 在前，`setTimeout2` 在后。

若将上述代码中的 `Promise.resolve().then` 改为 `queueMicrotask`，效果相同。若去掉 `Promise` 改用 `process.nextTick`（Node.js），其优先级高于 Promise 微任务，输出顺序会变为 `nextTick` 先于所有 Promise 回调。

### 4. 常见误区与进阶思考
常见误区 1：认为 `setTimeout(fn, 0)` 会立即执行，或认为它与其他宏任务按注册时间严格顺序执行。实际上，`setTimeout` 的 0 毫秒只是最短延迟，回调会进入宏任务队列，必须在当前宏任务结束后、所有微任务清空后才可能执行；且多个 `setTimeout` 的实际执行顺序还受定时器过期时间、宏任务队列中其他 I/O 任务影响，并非严格 FIFO。

常见误区 2：认为微任务在执行过程中若产生新的微任务，会推到下一轮事件循环。实际机制是“清空队列”，微任务队列中新增的微任务会继续在当前清空阶段执行，直到队列为空。因此，如果在微任务中无限递归添加微任务，主线程会永久阻塞，事件循环无法进入下一个宏任务，页面会卡死；而宏任务中递归添加宏任务则不会阻塞，因为每次只取一个宏任务，宏任务队列的推进依赖于事件循环的多轮迭代。

思考题：为什么在微任务中递归注册 `Promise.then` 会导致主线程卡死，而在宏任务（如 `setTimeout`）中递归注册自身却不会？请从事件循环的“取一个宏任务、清空整个微任务队列”这一根本调度规则出发，解释两者对队列长度的不同影响，并进一步说明：如果修改事件循环使宏任务也一次清空，会带来什么后果？
