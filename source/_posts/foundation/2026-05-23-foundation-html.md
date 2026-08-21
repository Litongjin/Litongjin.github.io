---
title: "每日基础技术总结 · 2026-05-23 · HTML 解析器中的预加载扫描器与脚本阻塞"
date: 2026-05-23 08:00:00
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-05-23 · HTML 解析器中的预加载扫描器与脚本阻塞

## 📚 今日主题

> **HTML 解析器中的预加载扫描器与脚本阻塞**（前端底层与计算机基础）

### 1. 核心概念速览
预加载扫描器（preload scanner）是HTML解析器内部的一个辅助机制，其核心作用是解决“脚本阻塞”导致的资源发现延迟问题。在标准HTML解析流程中，当主解析线程遇到同步脚本（无async/defer）时，必须暂停DOM构建，等待脚本下载并执行完成。这意味着脚本之后的所有资源（图片、样式、后续脚本）都无法被主线程发现，从而形成串行依赖。预加载扫描器作为独立于主解析线程的轻量级组件，在脚本阻塞期间继续读取剩余的字节流，识别其中的资源标签（如<link>、<img>、<script>等），并立即向网络层发出预加载请求。它不改变解析顺序，不执行脚本，不构建DOM，只做资源发现的加速。在整个Web性能体系中，它属于关键渲染路径优化的一部分，与DOM、CSSOM、网络栈紧密协作。专业工程师必须掌握它，因为实际生产中的首屏性能优化、资源优先级调度、以及脚本阻塞行为的分析都依赖于对这一机制的正确理解。

### 2. 底层原理剖析
底层机制：HTML解析器由主解析线程（main parser）和预加载扫描器（preload scanner）组成。主解析线程逐字节读取HTML，构建DOM树。当遇到<script>（非async/defer）时，主解析线程立即停止DOM构建，进入脚本获取与执行阶段。此时，预加载扫描器接管输入流，它不构建DOM，而是通过简单的令牌化（tokenization）扫描剩余的HTML，识别出带有资源属性的标签（例如<img src>、<link href>、<script src>），并同步发出资源请求。它不会执行脚本，也不会解析样式，因此开销极低。预加载扫描器持续扫描直到主解析线程恢复，之后两者协调工作。该机制与前端已知概念对比：类似于JS中的异步代理（如Web Worker）与主线程的关系——预加载扫描器是一个只读的、旁路的扫描器，不与主解析器共享解析状态，只共享输入流。它与async/defer的区别：async/defer是脚本执行时机的属性，而预加载扫描器不受这些属性影响，它始终尝试预加载任何可预加载的资源。伪代码逻辑如下：

function mainParser(htmlStream) {
  while (token = readNextToken()) {
    if (token is <script> and not async/defer) {
      preloadScanner.start(streamPosition);  // 启动预加载扫描器
      fetchAndExecuteScript(token.src);      // 主线程阻塞直到脚本执行完毕
      preloadScanner.stop();                 // 恢复主解析
    }
    buildDOM(token);
  }
}

function preloadScanner() {
  while (true) {
    token = peekNextToken(stream);
    if (token has src or href attribute) {
      networkLayer.requestResource(token.attributeValue);
    }
    advance(stream);
  }
}

预加载扫描器不会重复请求已经请求过的资源，它维护一个请求去重列表。同时，它需要正确处理HTML实体、注释、CDATA等边缘情况，但比主解析器更宽松，允许不完整的标签。

### 3. 基础代码与实战验证
```text
以下为验证预加载扫描器与脚本阻塞的最小HTML案例。将这段代码放在本地服务器中，打开浏览器的Network面板，观察资源加载时序：

<!-- index.html -->
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>Preload Scanner Test</title>
  <!-- 同步阻塞脚本，下载并执行需要一定时间（例如延迟200ms） -->
  <script src="slow.js"></script>
  <!-- 下面的资源本应被阻塞，但预加载扫描器会在slow.js执行期间提前请求它们 -->
  <link rel="stylesheet" href="style.css">
  <img src="image.png">
  <script src="after.js"></script>
</head>
<body>
  <h1>Hello</h1>
</body>
</html>

/* slow.js 内容：
   const start = Date.now();
   while (Date.now() - start < 300) { /* 空循环阻塞300ms */ }
*/

在Network面板中，如果预加载扫描器生效，你会看到：slow.js在时间轴开始处加载，随后style.css、image.png、after.js几乎同时开始下载，而不是等slow.js执行完后才开始。关键注释：
- <script src="slow.js"> 这一行触发主解析线程暂停。
- 预加载扫描器在暂停期间扫描到后续<link>、<img>和<script>，立即发出请求。
- 注意：预加载扫描器也会预加载after.js，但不会执行它，主线程仍然按顺序在slow.js之后执行after.js。

如果使用动态插入的脚本（如document.createElement('script')），预加载扫描器不会参与，因为动态脚本不在HTML字节流中，这体现其工作范围仅限原始输入流。
```

### 4. 常见误区与进阶思考
误区1：认为预加载扫描器会预加载所有资源。实际上，预加载扫描器只能扫描到主解析器尚未解析但已经出现在输入流中的内容。如果资源是动态插入的（通过JavaScript生成），预加载扫描器无法预知。另外，某些资源（如通过CSS加载的图片）不在HTML中，预加载扫描器也无法处理。因此，现代性能优化中需要用<link rel=preload>显式声明关键资源。

误区2：认为预加载扫描器会提前执行脚本或改变脚本执行顺序。它只负责发起网络请求，不会执行脚本，也不会改变主解析器的执行语义。脚本阻塞的规则完全不变，预加载扫描器只是将网络加载时间从总阻塞时间中剥离出来，使得脚本执行完成后资源已就绪，但解析和执行的顺序仍然严格遵循HTML规范。

进阶思考题：如果预加载扫描器在扫描过程中遇到了一个同步脚本，它是否会暂停自己的扫描？为什么？考虑多个同步脚本的嵌套场景，预加载扫描器如何避免请求重复或遗漏？请从实现层面解释其状态管理。
