---
title: "每日基础技术总结 · 2025-07-31 · 事件循环的渲染步骤与 requestAnimationFrame 时机"
date: 2025-07-31 20:00:00
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2025-07-31 · 事件循环的渲染步骤与 requestAnimationFrame 时机

## 📚 今日主题

> **事件循环的渲染步骤与 requestAnimationFrame 时机**（前端底层与计算机基础）

### 1. 核心概念速览
概念本质：渲染管线（Rendering Pipeline）与事件循环（Event Loop）在时间维度上的严格序列化机制。解决核心问题：确保用户交互响应、样式计算、布局、绘制等阶段在单一主线程中按确定性顺序执行，避免竞态条件导致的视觉撕裂或性能浪费。机制说明：浏览器将每帧（Frame）的处理划分为固定阶段（Style/Layout/Paint/Composite），并将脚本任务、宏任务、微任务及回调嵌入此流程。requestAnimationFrame (rAF) 的核心在于其回调被强制插入到『下一帧的 Paint 之前』，确保 DOM 操作后的视觉状态已收敛。为何必须掌握：它是理解高性能动画、首屏优化、虚拟DOM协调机制以及Web Workers同步通信边界的基石，直接决定应用对硬件刷新率（通常60Hz，即16.67ms/帧）的适配能力。

### 2. 底层原理剖析
底层运行机制（伪代码逻辑）：
while (hasPendingTasks || isAnimating) {
  // 1. Input Handling: 处理用户输入事件
  processInputEvents();

  // 2. Async Callbacks: 处理微任务和剩余宏任务
  while (!microtaskQueue.isEmpty()) {
    executeMicrotasks();
  }
  if (macrotaskQueue.isEmpty() && !needsAnimationFrame) break;

  // 3. Idle Callbacks (Optional): 低优先级任务
  if (TimeRemaining > threshold) runIdleCallbacks();

  // 4. RAF Execution: 关键时机点
  if (needsAnimationFrame) {
    for (let cb of rafCallbacks) {
      cb(performance.now());
    }
    rafCallbacks.clear();
  }

  // 5. Render Tree Construction & Layout (Reflow)
  // 此时所有 JS 执行的 DOM 变更已生效
  computeStyles();   // 样式计算
  layout();          // 几何布局
  paint();           // 绘制位图/路径
  compositeLayers(); // 合成图层并推送至 GPU

  // 6. Display Swap: 信号发送给显示器
  swapBuffers();
}

对比前端已有概念：
- 与 setInterval/setTimeout：后两者基于绝对时间戳触发，可能落在任意中间阶段（如正在重排时），导致掉帧；rAF 基于帧边界触发，具有自适应刷新率的同步性。
- 与 CSS Animation：CSS Animation 由合成器线程处理（GPU加速），无需 JS 介入布局；JS rAF 需经历完整的主线程样式-布局-绘制开销，适用于动态逻辑驱动的动画。

### 3. 基础代码与实战验证
```text
// 验证 rAF 在渲染周期的确切位置
// 关键点：DOM 修改后立即请求 rAF，回调执行时浏览器尚未提交绘制
function demonstrateRenderTiming() {
  const el = document.querySelector('#target');
  
  // 第一步：执行大量阻塞性计算或复杂 DOM 操作
  el.style.width = '50px'; // 修改样式，标记为需要重排
  requestAnimationFrame((timestamp) => {
    // 【核心验证】此处 timestamp 对应即将开始绘制的帧头
    // 在此刻调用 getBoundingClientRect() 是安全的且准确的
    // 因为上一轮宏任务结束前，所有的 Style/Layout 计算已完成
    const rect = el.getBoundingClientRect();
    console.log(`Frame at ${timestamp}ms, Element width: ${rect.width}`);
    
    // 若在此处再次修改 DOM，将触发下一帧的计算
    el.style.height = '100px'; 
  });

  // 第二步：微任务队列测试
  Promise.resolve().then(() => {
    // 微任务在当前宏任务末尾、rAF 回调前执行
    // 如果这里改 DOM，会合并进同一帧的 Layout 阶段
    el.style.color = 'red'; 
  });
}
```

### 4. 常见误区与进阶思考
误区纠正：
1. "rAF 总是比 setInterval 快"：错误。rAF 仅在屏幕刷新率高于定时器间隔时才显得"快"且流畅。在高负载导致掉帧（如从60fps跌至30fps）时，rAF 自动降低频率以节省资源，而 setInterval 仍按原频率堆积任务，导致更严重的卡顿。
2. "rAF 内部立即执行绘制"：错误。rAF 回调只是通知脚本"请准备数据"，真正的绘制发生在回调函数全部执行完毕、进入浏览器内部管线之后。如果在回调中抛出异常或无限循环，会导致当前帧无法完成绘制。

深度思考题：
当使用 Web Worker 通过 postMessage 向主线程发送 DOM 结构更新指令，同时主线程恰好处于 requestAnimationFrame 的执行阶段，Worker 的消息何时能影响下一帧的 Layout？这如何影响多线程架构下的渲染一致性设计？
