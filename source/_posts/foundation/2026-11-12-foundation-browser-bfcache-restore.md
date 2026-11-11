---
title: "每日基础技术总结 · 2026-11-12 · Browser Back/Forward Cache（bfcache）：页面缓存的生命周期与 restore 事件"
date: 2026-11-12 07:02:24
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-12 · Browser Back/Forward Cache（bfcache）：页面缓存的生命周期与 restore 事件

## 📚 今日主题

> **Browser Back/Forward Cache（bfcache）：页面缓存的生命周期与 restore 事件**（前端底层与计算机基础）

### 1. 核心概念速览
Back/Forward Cache（bfcache）是浏览器会话历史导航的文档级冻结缓存机制：当用户通过 back/forward 或可缓存的 history traversal 离开页面时，浏览器若判定该活动文档可缓存，则不销毁其 DOM Tree、CSSOM、JS Heap、Web API 状态与布局状态，而是将 Document 转入 frozen 生命周期并保留在 Session History 中；再次前进/后退时，直接从 frozen 状态 restore，跳过冷启动中的资源加载、HTML/CSS 解析、DOM/CSSOM 构建、脚本执行和首屏渲染。它解决的是历史导航延迟与状态丢失，而非资源复用；本质是页面生命周期状态机的内存驻留，而非 HTTP Cache、Service Worker 或 SPA 虚拟视图缓存。专业工程师必须掌握它，因为它改变前端初始化、事件注册、定时任务、网络请求、存储写入和错误恢复的默认假设：一个文档可能在已隐藏但仍存活与完全终止之间切换。

### 2. 底层原理剖析
核心机制建立在 Page Lifecycle 与 Session History 的交集上。
1. 离开阶段：文档因导航变为非活跃。浏览器使文档变为 hidden，并派发 pagehide。若文档满足缓存资格，Document 不进入 unload/terminated，而是被冻结为 frozen，并挂在对应 history entry 上。
2. frozen 阶段：DOM、CSSOM、JS Heap、闭包、class 实例、模块状态、已注册事件监听器仍在内存中，但页面不再接收常规交互。普通 setTimeout、setInterval、requestAnimationFrame 不再按原速率推进；事件循环被挂起或严格调度。部分后台任务可能被继续调度，但应用逻辑不能假定它们仍在线运行。
3. restore 阶段：用户 back/forward 命中可缓存 history entry。浏览器解冻 Document，将其重新加入可调度状态，派发 pageshow 且 event.persisted=true，随后 document.visibilityState 回到 visible。此时没有重新解析脚本或重建 DOM；JS 上下文连续，但 wall-clock、网络状态、服务器会话、用户权限、WebSocket 状态可能已变化。
4. 失效与淘汰：存在 unload 事件监听器、页面触发不可缓存条件、资源/安全上下文限制、刷新、内存压力、会话过期、DevTools 或平台策略，都可能使页面无法进入或无法恢复。恢复时若未命中 frozen 文档，则走普通导航，pageshow 的 persisted=false。
与前端既有机制对比：
- HTTP Cache 复用字节资源，不复用运行时状态；bfcache 复用执行后的文档状态，是生命周期缓存。
- Service Worker 提供请求拦截与可编程缓存，bfcache 不决定资源如何获取，只消除历史导航的启动成本。
- SPA KeepAlive、Redux/Vuex 状态缓存只保留当前 document 内的应用对象；bfcache 保留整个 document 上下文并可跨历史条目恢复。
- 这类似 Java 的 interface 提供语言运行时多态约束，TS 的 interface 是编译期静态抽象：bfcache 也不是业务层缓存 API，而是浏览器运行时对 Document 生命周期的系统级契约。

### 3. 基础代码与实战验证
```text
// 在页面中注入以下代码；用浏览器返回/前进观察事件顺序。
let hideState = null;

window.addEventListener('pagehide', event => {
  // event.persisted === true：浏览器离开本页面时选择保留文档，可能进入 bfcache；false：文档将 unload 并 terminate。
  // 此处可记录进入 hide 时的业务状态；注意即使 persisted=true，最终 restore 仍可能被内存回收或资格失效破坏。
  hideState = {
    persisted: event.persisted,
    time: performance.now(),
    scrollY: window.scrollY
  };
});

window.addEventListener('pageshow', event => {
  // event.persisted === true：从 bfcache restore，当前 JS 上下文仍是原上下文，不重新执行 script 标签。
  // event.persisted === false：冷启动或缓存未命中，需要重新初始化。
  if (event.persisted) {
    // restore 时冻结期间的 setTimeout 可能已错过，动画/WebSocket/订阅需要在此重新校验。
    console.log('bfcache restore', hideState, 'now', performance.now());
  }
});

window.addEventListener('visibilitychange', () => {
  // restore 后通常变为 visible；用 visibilitychange 比 pageshow 更适合同时处理 tab 切换与 bfcache 恢复。
  if (document.visibilityState === 'visible') {
    // 示例：若存在动画循环，应基于当前 timestamp 恢复而非假定上一帧仍在持续。
  }
});

// 验证步骤：
// 1. 打开 page A，设置一个每 1000ms 递增的 setTimeout 或 rAF。
// 2. 通过链接导航到 page B。
// 3. 浏览器点击 Back。
// 4. 观察 A 的 pagehide.persisted 与 pageshow.persisted；若均为 true，且计时器在隐藏期间未继续累加，即可证明 JS 上下文被冻结而非普通卸载。
```

### 4. 常见误区与进阶思考
常见误区：
1. 把 pagehide.persisted=true 当成必定从 bfcache 恢复。它只表示当前离开路径选择了保留文档，不代表后续不会因内存压力、会话过期、刷新、DevTools、资源加载状态或平台策略被丢弃。健壮实现必须同时处理 pageshow.persisted=false 的冷启动路径，以及恢复失败后的完整初始化路径。
2. 把 bfcache restore 当成页面从未离开。bfcache 保留 JS Heap 与 DOM，但时间、网络、权限、服务器会话可能已变化；冻结期间 rAF/普通定时器不推进，WebSockets 可能断开，fetch 结果可能延迟返回。必须将 pageshow.persisted=true 视为带旧状态的续命运行，需要重新校验连接、订阅、表单焦点、滚动位置和过期数据。
进阶思考：
若页面同时使用流式 fetch、ReadableStream reader、AbortController、WebSocket 和 IndexedDB 事务，在 bfcache freeze/restore 后如何设计状态机，使网络流在冻结期间被终止但 Promise 可能迟到 resolve、用户前进后仍看到旧页面、服务器会话已过期三者可区分？需要从 Document 生命周期、任务队列恢复、stream reader 锁、abort signal 语义和 UI 一致性五个层面给出精确恢复顺序。
