---
title: "每日基础技术总结 · 2026-09-27 · 浏览器主线程上的任务、帧与渲染步骤的时序关系"
date: 2026-09-27 07:03:48
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-27 · 浏览器主线程上的任务、帧与渲染步骤的时序关系

## 📚 今日主题

> **浏览器主线程上的任务、帧与渲染步骤的时序关系**（前端底层与计算机基础）

### 1. 核心概念速览
浏览器主线程上运行着三种关键时序单元：任务（Task）、帧（Frame）和渲染步骤（Rendering Steps）。任务是事件循环中不可中断的原子执行单元，包括用户事件回调、setTimeout/setInterval回调和I/O回调；帧是显示器输出画面的一次更新周期，由垂直同步信号（vsync）驱动，通常约16.7ms；渲染步骤是在帧开始后按序执行的样式计算、布局、绘制和合成过程。它们的时序关系是：主线程在一个周期内反复从任务队列取出任务执行，过程中清空微任务队列，并在满足渲染时机时执行一次渲染步骤。该机制定义了单线程事件循环如何协调“逻辑计算”和“视觉输出”，是前端性能优化（如避免长任务、防止布局抖动）的理论基石。专业工程师必须掌握它，因为所有交互延迟、掉帧和滚动卡顿的根本原因都归结为任务和渲染步骤之间的竞争关系。

### 2. 底层原理剖析
底层事件循环的精确模型（参考HTML规范）可抽象为以下伪代码：

    while (true) {
        task = taskQueue.takeNext();
        task.run();                       // 同步执行整个任务，期间不响应vsync
        microtaskQueue.runAll();          // 清空微任务队列（如Promise回调）
        if (now >= nextRenderTime) {      // 时间戳达到vsync边界且需要渲染
            // 更新渲染步骤
            runAnimationFrameCallbacks(); // requestAnimationFrame回调
            resize / scroll 事件处理；
            styleAndLayout();             // 计算样式与布局
            paint();                      // 生成绘制指令
            composite();                  // 合成图层提交到GPU
        }
    }

关键点：
- 每个任务和其微任务组成一个不可分割的单位；微任务永远在任务返回前清空，因此微任务过度递归会饿死渲染。
- 渲染不是每次任务后都发生，而是由浏览器根据vsync、运行环境和帧预算决定。显示刷新率（如120Hz）会影响 now >= nextRenderTime 的判定。
- 任务队列中的旧任务可能阻塞渲染；若任务执行时间超过帧余量，则对应的vsync会完全错过，表现为掉帧。
- requestAnimationFrame（rAF）不是任务，而是渲染步骤的一部分；它排在样式计算之前，因此回调内修改DOM不会触发强制同步布局（前提是回调内不读取几何值）。
- 与前端已有的“事件循环宏/微任务”知识相比，此处新增了“渲染”这一更高层级的阶段：宏任务和微任务处理的是数据状态更新，而渲染步骤将状态映射为像素；宏观上，宏任务→微任务→渲染构成一个更大的周期，但该周期并不严格每轮迭代都执行渲染。

此模型的本质是：单线程事件循环内部存在一个二阶段调度——先完成逻辑（任务+微任务），再决定是否进行视觉呈现（渲染）。

### 3. 基础代码与实战验证
```text
极简验证（在浏览器控制台执行，并使用 Chrome DevTools Performance 面板观察）：

    const t0 = performance.now();
    console.log('1. task start', t0);

    // 微任务在当前 task 结束前清空
    Promise.resolve().then(() => console.log('2. microtask', performance.now() - t0));

    // 宏任务排入队列，等待下一个事件循环迭代
    setTimeout(() => {
        console.log('3. timeout task', performance.now() - t0);
        // 在这个任务里再次请求 rAF，看它是否在本帧渲染前被调用
        requestAnimationFrame(() => console.log('4. rAF from timeout', performance.now() - t0));
    }, 0);

    // 请求 rAF：它将在本帧（或最近可渲染帧）的渲染步骤中被调用
    requestAnimationFrame(() => console.log('5. rAF first', performance.now() - t0));

    // 强制同步布局：这里会立即执行 layout（如果此前有脏样式）
    document.body.offsetWidth;
    console.log('6. task end', performance.now() - t0);

顺序推论：
- 第1步到第6步是同一个宏任务，期间主线程不可被打断。
- 第2步的微任务必然在第6步之后执行，且仍在同一宏任务内。
- 第3步的宏任务与第5步的 rAF 的执行先后取决于当前任务结束后的时间是否达到下一次渲染时机。如果已达到，则 rAF 优先；否则先执行宏任务。
- 第4步嵌套的 rAF 会等到下一次渲染步骤再执行。
- 最后一步强制读取 offsetWidth 会导致同步布局，如果此前有样式变更，它把本应放在渲染步骤中的 layout 工作提前到当前任务内执行，这是一种典型的强制同步布局（forced synchronous layout）。
```

### 4. 常见误区与进阶思考
误区1：认为每次宏任务后都会发生一次渲染。实际渲染由 vsync 决定，且可能连续执行多个宏任务后才渲染一次；反之，如果任务占满整个帧，渲染会被跳过。
误区2：将 requestAnimationFrame 当作宏任务。它不在任务队列中，而是由渲染步骤调度；也因此，rAF 回调里修改样式不容易触发强制同步布局（除非读取几何值），但微任务和宏任务中的 DOM 修改往往会造成布局抖动。

思考题：一个 setTimeout(fn, 0) 和一个 requestAnimationFrame(fn) 同时注册，为什么浏览器可能先执行 setTimeout 而不是 rAF？请结合事件循环中“渲染时机”的判断条件，说明什么情况下 rAF 回调会晚于下一个宏任务执行。
