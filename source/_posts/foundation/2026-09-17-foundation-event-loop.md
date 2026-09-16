---
title: "每日基础技术总结 · 2026-09-17 · Event Loop"
date: 2026-09-17 07:01:29
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-17 · Event Loop

## 📚 今日主题

> **Event Loop**（前端底层与计算机基础）

### 1. 核心概念速览
Event Loop（事件循环）是**宿主环境**（Browser / Node.js / Deno / Bun）实现的、驱动单线程 JavaScript 引擎持续消费异步任务的控制流调度循环。关键前提：它不属于 ECMAScript 规范。ECMAScript 只定义了执行上下文栈、Execution Context Stack、以及 Job Queue（微任务队列）与 RunJobs / HostEnqueuePromiseJob 等抽象操作；真正的「循环」由 HTML Standard 的 Event Loop Processing Model、以及 Node.js 侧的 libuv 事件循环定义。

**它解决什么问题**：JS 引擎只有一个执行线程、一个调用栈，所有同步执行都是 run-to-completion（运行到完成、不可被中断）的。若没有 Event Loop，setTimeout、网络 I/O 完成、用户事件这些由外部（内核/宿主）产生的回调就没有任何合法的执行时机，且任何阻塞式 I/O 都会冻结整个线程。Event Loop 的机制是：把外部完成事件序列化进 FIFO 任务队列，并在**调用栈为空**且**微任务队列为空**时取出下一个任务压栈执行，从而让单线程获得并发（concurrency）能力——注意，是并发，不是并行（parallelism）。

**在整个体系中的位置**：它是 JS 运行时与操作系统内核之间的适配层。真正的并行由内核提供（硬件中断、多路复用 select/epoll/kqueue/IOCP、线程池），Event Loop 负责把这些「就绪」信号转化为可执行的任务序列。因此 Event Loop 是 I/O 模型的一部分，而不仅是语言特性。

**为什么必须掌握**：所有异步正确性问题（Promise 续体时机、React 18 批处理与并发渲染、Vue 的 nextTick）、所有性能问题（长任务、掉帧、INP/TBT 指标）、所有服务端吞吐模型（Node 的 C10K 解法、Netty EventLoop、Redis 单线程模型）最终都收敛到 Event Loop 的调度语义。不掌握它，就无法解释任何一段异步代码的执行顺序，也无法解释「为什么这里会掉帧」。

### 2. 底层原理剖析
## 一、浏览器的 Event Loop Processing Model

规范层面可精确抽象为如下循环（伪代码，按 HTML Standard 简化）：

    loop:
      task = pickOldestRunnableTask(taskQueues)   // 队列集合由实现定义，如 timers / I-O / UI 事件
      if (task != null) {
          currentTask = task
          invoke(task.callback)                   // 期间产生的微任务仅入队，不执行
          currentTask = null
      }
      performMicrotaskCheckpoint()                // 关键：清空直到队列为空，包括执行中新增的
      if (hasRenderingOpportunity()) {            // 由 VSync 驱动，与刷新率对齐
          runAnimationFrameCallbacks(timestamp)   // requestAnimationFrame 回调
          updateRendering()                       // style -> layout -> paint -> composite
      }
      goto loop

三个必须记住的规范要点：
1. **microtask checkpoint 的语义是「drain until empty」**，不是「执行一轮」。在 checkpoint 执行期间由回调新压入的微任务，仍属于本轮 checkpoint，必须在同一个 task 与下一个 task 之间被消费干净。
2. **rendering opportunity 不是每个 task 之后必然发生**，它由 VSync 触发；如果 event loop 被长任务占满，checkpoint 之后的渲染步骤会被跳过，表现为掉帧。
3. **requestAnimationFrame 既不是宏任务也不是微任务**，它属于 rendering step，因此它的执行时机在微任务清空之后、layout 之前。

## 二、任务来源

宏任务（task）：初始 script 求值、setTimeout/setInterval、MessageChannel/MessagePort、I/O 完成回调、UI 事件回调（click 等与 task 绑定）、requestIdleCallback（空闲期）。

微任务（Job）：Promise 的 then/catch/finally 回调、await 之后的续体、queueMicrotask、MutationObserver 回调（宿主定义）、以及 Node 的 process.nextTick（独立队列，优先级高于 Promise 队列）。

## 三、Node.js 侧：libuv phases

Node 不是单一队列循环，而是 libuv 的阶段式循环：

    while (loop alive) {
      timers              // 到期 setTimeout / setInterval
      pending callbacks   // 上一轮延迟的 I/O 回调
      idle, prepare
      poll                // 阻塞等待 I/O，核心阶段
      check               // setImmediate
      close callbacks     // socket.destroy 等
      // 每个 phase 的每个回调之间都会清空 nextTick 队列，再清空 Promise 微任务队列
    }

## 四、与前端/AI 工程师已有知识体系的对比

- **vs Java 多线程**：Java 是抢占式调度，由 OS 时间片中断与 JVM 锁（synchronized / AQS）保证内存可见性与互斥。Event Loop 是非抢占式协作调度，任务一旦开始不会被打断，因此 JS 中不存在数据竞态——代价是任何长任务直接阻塞所有后续工作与渲染。
- **vs Netty 的 EventLoop**：二者同构。Netty EventLoop = 单线程 + Selector + MPSC taskQueue，run() 的形态是 loop { select(); processSelectedKeys(); runAllTasks(); }。差别在于 Netty 把「就绪 I/O」显式建模为 selectedKeys，而浏览器把它抽象成任务队列的一个来源；且 Netty 没有渲染阶段。
- **vs CompletableFuture**：JVM 的 future.thenApply 默认在「完成它的那个线程」上执行回调，这是一个隐式的线程跳转；而 JS 的微任务回调必然在调用栈清空后的 checkpoint 上执行。这是两套完全不同的执行时机契约。
- **vs 类型系统（TS interface vs Java interface）**：Event Loop 是运行时的调度契约，不是类型契约，无法被静态检查，只能通过执行时序来验证；它更接近 JMM（Java 内存模型）的地位——一套规定「可观察行为」的运行时规范。

## 五、执行顺序推演的本质规则

同一轮内：同步代码（当前 task） > 微任务 checkpoint（FIFO，含嵌套） > 渲染机会 > 下一个 task。
跨轮次：不同宏任务之间至少间隔一次完整 checkpoint 与（可能的一次）渲染步骤。因此 Promise 永远比 setTimeout 早，不是因为优先级高，而是它们处在执行流程的不同阶段。

### 3. 基础代码与实战验证
```text
// ===== 验证一：单轮内的执行阶段序列 =====

// ① 同步执行，仍处于「初始 script」这个 task 的调用栈内
console.log('A script start');

// ② 进入 timers 任务队列，成为后续某一轮的宏任务；延迟会被 clamp（嵌套 >= 5 层时最小 4ms）
setTimeout(() => console.log('G timeout'), 0);

// ③ 通过宿主 API 直接向 ECMAScript Job Queue 入队，FIFO
queueMicrotask(() => console.log('D queueMicrotask'));

// ④ .then 的回调经 HostEnqueuePromiseJob 进入同一个微任务队列，排在 ③ 之后
Promise.resolve().then(() => {
  console.log('E then');
  // ⑤ 在 checkpoint 执行期间新入队的微任务，仍由本轮 checkpoint 消费，不会漏到下一个 task
  queueMicrotask(() => console.log('F nested microtask'));
});

// ⑥ IIFE 与下一行仍是同步代码，栈未清空前不会执行任何回调
(() => console.log('B iife'))();
console.log('C script end');

// 实际输出：A B C D E F G
// 原因：D/E/F 全部位于「初始 script task 结束 → checkpoint 清空」这一区间，早于 G 所在的下一轮 task


// ===== 验证二：调用栈不清空 ⇒ 微任务与渲染都无法推进 =====

const t0 = performance.now();
while (performance.now() - t0 < 200) {}   // 长任务：占用调用栈 200ms
// 此期间 setTimeout 回调不执行、Promise.then 不执行、rAF 不触发、样式不重算不重绘
// 这就是 INP / TBT 恶化的直接成因：event loop 无法进入 checkpoint 与 rendering step


// ===== 验证三：渲染机会与 rAF 的归属 =====

let frame = 0;
function tick() {
  frame++;
  if (frame < 10) requestAnimationFrame(tick);
}
requestAnimationFrame(tick);
// rAF 回调在 rendering step 内执行（微任务清空之后、layout 之前），与 VSync 对齐
// 因此 rAF 内做 DOM 写入是最优时机：读操作已被上一帧结束，写操作会合并进本帧的 layout


// ===== 验证四：Node.js 的微任务优先级差异 =====

setImmediate(() => console.log('check phase'));
setTimeout(() => console.log('timers phase'), 0);
Promise.resolve().then(() => console.log('promise microtask'));
process.nextTick(() => console.log('nextTick'));
// 主模块中输出顺序：nextTick → promise microtask → timers phase / check phase（后两者相对顺序在主模块内不确定，
// 取决于进程启动耗时是否超过 1ms；置于 I/O 回调内则 setImmediate 恒定先于 setTimeout）
// 本质：nextTick 队列独立于 Promise 微任务队列，且在每个 phase 的每个回调之间先于后者被清空
```

### 4. 常见误区与进阶思考
## 误区一：认为「微任务优先级高于宏任务」

规范层面不存在跨队列的优先级比较，微任务队列与任务队列不是同一个调度器里的两个优先级。正确的模型是**阶段顺序**：每个 task 结束后必然执行一次 microtask checkpoint 并将其清空，然后才可能进入 rendering step，最后才取下一个 task。所谓「优先」只是这个阶段序列的副产物。一旦接受错误模型，就会出现「用 queueMicrotask 拆长任务来避免掉帧」这类失效优化。

## 误区二：认为 await 会让出线程 / 是异步的

await expr 中 expr 是**同步求值**的，只是 await 之后的续体（continuation）被封装为 Job 压入微任务队列，等价于 expr.then(rest)。因此 await 之前的代码同步执行；即使 await 一个非 Promise 的值，仍会产生一次额外的 microtask tick。这也解释了为什么 for-await 循环中每次迭代都会让出一次 checkpoint，而普通 for 循环不会。

## 误区三：认为微任务可以用来让出主线程

这是最反直觉的一条：microtask checkpoint 必须「清空直到为空」才会到达 rendering step。因此用 queueMicrotask 递归拆分工作，永远无法让浏览器渲染，只会无限推迟渲染。要真正让出渲染，必须使用宏任务边界（setTimeout、MessageChannel、scheduler.postTask），或 requestIdleCallback / rAF 分帧。

## 思考题

给定如下代码，请判断页面状态与 event loop 的运行状态，并解释机制：

    function loop() { Promise.resolve().then(loop); }
    loop();

追问三点：
1. 任务队列是否还在被消费？渲染机会是否还会到来？页面主线程为何完全失去响应（包括输入事件与 rAF）？
2. 若把 Promise 换成 setTimeout(loop, 0)，行为有什么本质差异？为什么后者不会立即冻结页面，但最终仍会崩溃？
3. 基于同一机制，解释为什么 React Scheduler 在浏览器端选择 MessageChannel 而不是 Promise 或 setTimeout 作为时间切片的时间源。
