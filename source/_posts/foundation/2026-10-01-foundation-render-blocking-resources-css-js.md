---
title: "每日基础技术总结 · 2026-10-01 · 渲染阻塞资源：CSS 与脚本在关键渲染路径中的行为"
date: 2026-10-01 07:04:22
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-01 · 渲染阻塞资源：CSS 与脚本在关键渲染路径中的行为

## 📚 今日主题

> **渲染阻塞资源：CSS 与脚本在关键渲染路径中的行为**（前端底层与计算机基础）

### 1. 核心概念速览
渲染阻塞资源（Render-Blocking Resource）是指浏览器在完成首次渲染（First Paint）之前，必须下载并解析的 CSS 或 JavaScript 资源。其本质是关键渲染路径（Critical Rendering Path）上的同步依赖：DOM 和 CSSOM 必须都就绪才能生成渲染树，而普通脚本会因为可能修改 DOM/CSSOM 而在执行前被强制等待 CSSOM 构建完成。该机制解决的核心问题是如何在资源加载与解析执行之间建立正确的顺序约束，以保证渲染的正确性和一致性。在计算机体系结构中，该知识点位于浏览器渲染引擎（Blink/WebKit）的 HTML 解析器与样式系统、脚本系统之间的协作层，是网络资源获取（Network）与渲染管线（Rendering Pipeline）之间的关键契约。专业工程师必须掌握它，因为所有首屏性能优化（如 critical CSS、async/defer、preload）本质上都是如何打破或推迟这条同步依赖链。

### 2. 底层原理剖析
HTML 解析器（HTMLParser）逐 token 处理。遇到 <link rel='stylesheet'> 时，将其加入样式表列表，并继续解析 DOM；但渲染会被推迟。遇到普通 <script>（无 async/defer）时，解析器会先检查是否有未加载完的样式表：若有，则暂停解析，等待样式表下载完成并构建出全局 CSSOM 后才开始获取脚本；这是因为脚本可能通过 getComputedStyle 等 API 读取样式。随后脚本下载并同步执行，执行完毕后再恢复 HTML 解析。因此 CSS 虽然不直接阻塞 DOM 解析，但通过阻塞后续脚本而间接阻塞解析；而脚本则直接阻塞解析，解析停顿自然阻塞渲染。其状态机可用伪代码表示为：

  loop {
    token = parser.next()
    if (token is link stylesheet) {
      pendingStylesheets.push(token)
      continue
    }
    if (token is script && !token.async && !token.defer) {
      waitUntil(pendingStylesheets.allLoaded)   // CSSOM ready
      downloadAndExecute(token)                 // 阻塞解析
      continue
    }
    if (token is end marker) break
  }

渲染取决于 DOM + CSSOM 是否完整；只要任意脚本尚未执行，DOM 就不会完成，渲染自然不发生。与前端已有概念对比：这类似于 JS 事件循环中的任务队列依赖关系，但更贴近的是 Web API 中的依赖等待——样式表是脚本执行的前置互斥条件。CSS 与脚本的阻塞关系本质上是一种“生产者-消费者”同步，CSSOM 是共享缓冲区，脚本是消费者，HTML 解析器是调度器。

### 3. 基础代码与实战验证
```text
实验：观察 CSS 对普通脚本的阻塞。

slow.css（模拟 3 秒延迟）：
  body { background: red; }

test.html：
  <!DOCTYPE html>
  <html>
  <head>
    <link rel='stylesheet' href='slow.css'>
  </head>
  <body>
    <script>
      console.log('A: script start');
      console.log('B: stylesheets count =', document.styleSheets.length);
      console.log('C: body background =', getComputedStyle(document.body).backgroundColor);
      console.log('D: script end');
    </script>
    <p>hello</p>
  </body>
  </html>

底层行为：
  - 解析器读到 link 后发起 CSS 请求，继续解析。
  - 遇到 script 时，发现 pendingStylesheets 未清空，立即暂停解析，等待 slow.css 完成。
  - slow.css 到达后，构建 CSSOM，然后执行该内联脚本（此时 document.styleSheets.length >= 1，getComputedStyle 可读到背景色）。
  - 脚本执行完毕，解析器继续解析 p 标签，最终触发 DOMContentLoaded。
```

### 4. 常见误区与进阶思考
误区一：认为“CSS 只阻塞渲染，不阻塞解析”。实际上 CSS 不直接阻塞 DOM 解析，但会阻塞所有后续脚本的执行；而脚本会阻塞解析，因此 CSS 间接阻塞了解析。在无脚本的页面中，CSS 确实只阻塞渲染；一旦存在普通脚本，CSS 就变成了解析阻塞链的一环。

误区二：认为给 script 加上 async 后就不会被 CSS 阻塞。async 只是让下载不阻塞解析，但执行仍需要等待 CSSOM 构建完成，因为任何脚本在执行时都可能访问样式信息；所以在 CSS 未加载完成时，即使脚本已经下载完毕也不会执行。

思考题：假设页面上有 <link rel='stylesheet' href='slow.css'>，随后是 <script src='a.js'></script>，再随后是 <script src='b.js'></script>（均无 async/defer）。如果 slow.css 加载需要 3 秒，a.js 加载需要 5 秒但可以提前开始下载吗？b.js 又在何时发起请求？请描述浏览器在这一场景中的完整时序，特别关注 CSSOM 就绪、脚本下载排队和解析暂停之间的先后关系。
