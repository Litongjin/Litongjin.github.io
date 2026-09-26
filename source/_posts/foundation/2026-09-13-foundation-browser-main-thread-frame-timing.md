---
title: "每日基础技术总结 · 2026-09-13 · 浏览器主线程上的任务、帧与渲染步骤的时序关系"
date: 2026-09-13 08:00:00
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-13 · 浏览器主线程上的任务、帧与渲染步骤的时序关系

## 📚 今日主题

> **浏览器主线程上的任务、帧与渲染步骤的时序关系**（前端底层与计算机基础）

### 1. 核心概念速览
浏览器主线程上的任务（task）、帧（frame）与渲染步骤（rendering steps）是同一事件循环（event loop）中按严格时序衔接的三个层级的执行单元。任务指通过任务队列（task queue）调度的独立执行块，如脚本片段、setTimeout回调、事件回调；帧指一次屏幕刷新周期，通常与显示器刷新率（60Hz/120Hz）对齐；渲染步骤指在帧内执行的一组管线段，包括输入事件分发、requestAnimationFrame（rAF）回调、样式计算（style recalc）、布局（layout）、绘制（paint）与合成（composite）。其核心机制是：事件循环每轮只从任务队列取一个任务执行，执行完后若满足渲染时机（如当前时间接近垂直同步信号，或任务耗时长导致帧过期），则进入渲染步骤，而非每执行完一个任务就立即渲染。该机制解决的核心问题是：让 JavaScript 的异步逻辑与屏幕刷新节奏同步，避免一帧内多次布局/绘制导致性能浪费，以及避免屏幕撕裂。在整个计算机体系结构中，它位于浏览器内核的渲染进程与操作系统显示服务的交界处，是连接 Web API、JavaScript 执行与光栅化合成器的中间调度模型。专业工程师必须掌握它，因为这是一切前端性能优化（如避免强制同步布局、合理使用 rAF、避免长任务）的底层依据，也是理解响应式系统如何与帧对齐、如何测量 FPS 的基础。

### 2. 底层原理剖析
浏览器事件循环的底层时序可抽象为以下伪代码，直接来自 HTML 规范中的 processing model：

while (true) {
  // 1. 从任务队列中选择一个最老的任务执行
  const task = taskQueue.pop()();

  // 2. 执行微任务检查点（microtask checkpoint），将 microtask 队列清空
  microtaskQueue.drain();

  // 3. 判断是否需要渲染（update the rendering）
  if (needRender(timeNow)) {
    // 一组帧内步骤，严格顺序如下：
    // a. 处理用户输入事件（如 click、scroll）
    // b. 执行 requestAnimationFrame 回调（按注册顺序）
    // c. 样式重新计算（style calc）
    // d. 布局（layout 或 reflow）
    // e. 绘制（paint 或 rasterize）
    // f. 合成层合并（composite）并提交到 GPU
    render();
  }
}

关键点在于“渲染时机”并非每个任务后都发生。浏览器会记录当前帧的开始时间，并在每轮循环末尾检查是否已到下一帧的垂直同步点（vsync）。如果未到，则继续执行下一个任务；如果到了或已经超过，则执行渲染。这意味着：一个长任务会阻塞渲染，导致掉帧；而任务很短时，一个帧内可能执行多个任务。

与前端已有概念的对比：它本质上类似于“生产者-消费者”与“临界区”的组合。任务队列是生产者放入的，事件循环是消费者，而渲染步骤是必须持锁（帧期）执行的临界区。这与 Java 接口和 TypeScript 接口的对比类似：二者都定义了“契约”，但一个关注运行时多态，一个关注编译期结构；同样，任务与渲染虽然在事件循环中共存，但一个关注逻辑执行，一个关注像素输出，二者由“帧同步”这一隐式契约连接。理解这种异同，能避免把 rAF 当作 setTimeout 的替代品这类的认知偏差。

更深入地看，microtask 在渲染前会被完全清空，因此使用 Promise 或 MutationObserver 产生的回调会阻塞渲染，可能延迟帧提交；而使用 task 则可能被渲染步骤“插队”，导致回调时机不确定。rAF 是唯一一个由渲染系统主动调度、保证在下一帧样式计算前执行的回调，因此它适合批量读取布局和修改样式。而 setTimeout(fn, 0) 的最小延迟受限于当前任务队列和渲染调度，实际执行时间不精确，且如果前一轮有渲染步骤，则 setTimeout 的回调总在下一帧渲染之后才可能执行。

### 3. 基础代码与实战验证
```text
// 验证主线程任务、微任务、渲染帧与 rAF 的执行顺序
// 运行环境：浏览器，非 Node.js

// 1. 同步阶段：当前任务（script 标签）开始执行
console.log('1. script start');

// 2. 注册一个微任务，它会在当前任务结束后、渲染前执行
Promise.resolve().then(() => {
  console.log('2. microtask');
});

// 3. 注册一个 rAF 回调，它只会在下一帧的渲染步骤内执行
requestAnimationFrame(() => {
  console.log('4. rAF callback (before style/layout/paint)');
});

// 4. 注册一个 setTimeout 回调，它作为新任务被放入任务队列
setTimeout(() => {
  console.log('5. setTimeout task (after render in next event loop)');
}, 0);

// 5. 强制当前任务占用主线程超过一帧（例如 20ms，60Hz 下帧间隔约 16.7ms）
const start = performance.now();
while (performance.now() - start < 20) {
  // 阻塞主线程，模拟长任务
}

// 6. 当前任务结束，打印同步日志
console.log('3. script end');

// 预期输出顺序（在大部分浏览器中稳定）：
// 1. script start
// 3. script end
// 2. microtask
// 4. rAF callback (before style/layout/paint)
// 5. setTimeout task (after render in next event loop)

// 注释：
// - setTimeout 回调虽然延迟为 0ms，但需要等待浏览器完成至少一次渲染步骤后，
//   才会作为新任务被取出执行，因此它几乎总是出现在 rAF 之后。
// - microtask 在当前任务结束后立刻清空，不会等到渲染步骤，
//   所以它先于 rAF。
// - rAF 回调在浏览器渲染管道的最前端执行，此时尚未进行样式计算和布局。

// 扩展验证：连续观察 rAF 的调用频率可测量实际帧率
let last = performance.now();
function frame(time) {
  const delta = time - last;
  last = time;
  console.log('Frame interval:', delta);
  requestAnimationFrame(frame);
}
requestAnimationFrame(frame);
```

### 4. 常见误区与进阶思考
认知误区一：认为 setTimeout(fn, 0) 会在下一帧渲染之前执行。实际上，setTimeout 回调是 task，渲染步骤也是从事件循环中触发的独立流程。如果当前帧的 vsync 时机先到达，渲染会先于下一个 task 执行；反之如果任务队列清空后还没到 vsync，setTimeout 可能会执行。因此 setTimeout(fn, 0) 的时机与帧边界没有严格保证，而 rAF 则保证在渲染前执行。

认知误区二：认为每次修改 DOM 样式都会立即触发一次 layout，然后把避免强制同步布局仅仅理解为“少读多写”。事实上，布局本身是按需计算的，浏览器会懒执行。只有当主线程执行渲染步骤或同步读取布局属性（如 offsetWidth、getBoundingClientRect）时才会计算。而任务执行期间的多次样式修改在进入渲染步骤前只会维护 dirty 标记，最终合并为一次 layout。真正的性能杀手是在同一任务中先写后读，强制浏览器提前执行 layout，导致帧内出现多次布局。

深度思考题：假设一个动画帧内需要更新 100 个 DOM 节点的样式，然后用 100 次读取每个节点的 offsetWidth 来验证位置，最后再执行一个 ajax 回调。如果你把读取操作放进微任务而不是同步执行，是否可能避免强制同步布局？请结合事件循环和 microtask checkpoint 在每任务结束时的执行时机，分析微任务中的读取发生在渲染管道的哪一阶段，这能否减少布局次数？这直接检验你对“任务、微任务、渲染”三层时序的真正理解。
