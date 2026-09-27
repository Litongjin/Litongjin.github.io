---
title: "每日基础技术总结 · 2026-09-28 · 前端首屏性能优化体系（SSR/预加载/分包）"
date: 2026-09-28 07:04:39
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础（扩展）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-28 · 前端首屏性能优化体系（SSR/预加载/分包）

## 📚 今日主题

> **前端首屏性能优化体系（SSR/预加载/分包）**（前端底层与计算机基础（扩展））

### 1. 核心概念速览
前端首屏性能优化体系是围绕「关键渲染路径（Critical Rendering Path）」的一组工程手段，其核心目标是在最短时间内完成首屏内容的发现、获取、解析与绘制。SSR（Server-Side Rendering）本质上是将渲染环境从客户端浏览器迁移到服务端，使首包字节（TTFB）内即可携带可解析的 HTML 标记，绕过 JS 下载与执行的硬依赖；预加载（Preload/Prefetch）本质上是将资源发现时机从「解析器扫到引用节点」提前到「HTML 解析早期或响应头」，利用声明式标签显式告诉浏览器哪些资源优先级高；分包（Code Splitting）本质上是利用模块系统异步边界，将首屏所需的 JavaScript 体积切到最小，按需拉取后续功能块。三者的本质分别是解决「内容何时到达」「资源何时被发现」「代码何时被执行」三个独立瓶颈。在计算机系统视角中，这对应了缓存局部性、预取（Prefetch）和惰性求值（Lazy Evaluation）思想。专业工程师必须掌握其底层差异，才能在具体场景中正确组合，因为任何单一手段的误用都会引入新的延迟（如 SSR 增加 TTFB、滥用 Preload 抢带宽、过度分包增加请求数）。

### 2. 底层原理剖析
1. SSR 的底层机制：
- 当浏览器向服务端发起导航请求后，服务端执行组件到 HTML 字符串的序列化。以 React 为例，react-dom/server 的 renderToString 会将组件树映射为字符串。这个字符串包含完整的 DOM 结构与文本内容，浏览器 HTMLParser 在收到响应后即可逐步构建 DOM，无需等待任何 JavaScript 下载。
- 但 SSR 产生的 DOM 没有事件与状态，客户端需要以 hydrate 方式复用服务端 DOM 并绑定事件。因此，SSR 并没有消除 JS 执行，只是将「首屏标记的生成」从客户端执行栈移到了服务端 CPU。这也是为什么 SSR 对 TTI 的收益是间接的：它让 FP/FCP 提前，但 TBT（Total Blocking Time）可能因 hydration 脚本而增大。
- 对比前端已有概念：传统 JSP/PHP 也是服务端输出 HTML，但那是纯模板拼接；SSR 增加了一个抽象层——服务端与客户端共享同一组件树，且水合必须精确对齐，否则会造成 DOM 不匹配与恢复失败。

2. 预加载的底层机制：
- 浏览器默认的资源加载是由解析驱动的：当 HTMLParser 遇到 `<img>`、`<script>`、`<link>` 等标签时才发起对应请求。若某个关键 CSS/JS 位于文档中部甚至外部，那么首屏会经历两轮网络往返。`<link rel='preload'>` 放在 `<head>` 中时，浏览器会立即以该资源类型的预设优先级发起请求，无论它是否马上被使用。它本质上是将资源发现的时间点前提。
- 优先级由 `as` 属性确定：`as='style'` 会获得 CSS 的高优先级，`as='font'` 也会获得高优先级。若 `as` 声明错误，浏览器会忽略 preload 响应，导致二次加载。服务端可以通过 `Link: </style.css>; rel=preload; as=style` 响应头在 HTML 传输前就发起预加载，这是 HTTP 层的扩展。
- preload 与 prefetch 的区别：preload 是当前导航必需的资源，prefetch 是下一个导航可能需要的资源，优先级最低且只在网络空闲时下载。

3. 分包的底层机制：
- 模块打包器（Webpack/Rollup）在构建时从入口模块开始构建依赖图。当代码中出现 `import('./foo')` 时，该导入会被分离为一个独立的 chunk，并生成一个运行时加载函数。浏览器执行到 `import()` 时，运行时会动态创建 `<script src='chunk.js'>` 标签或使用 fetch 获取，并返回一个 Promise。
- 分包的价值在于减少首屏脚本体积。但脚本体积减少未必意味着更快的首屏：浏览器加载和执行脚本可能不是瓶颈，而 HTTP 连接数和服务器延迟可能是瓶颈。所以分包往往与 preload 搭配：预加载当前路由需要的首个异步 chunk。
- 对比前端已有概念：分包类似于「懒加载」，但 MPA 多页面应用是用页面切换完成按需加载，而 SPA 分包是在同一文档内按需拉取 chunk，没有文档重载。

### 3. 基础代码与实战验证
```text
// 运行环境：Node.js LTS
const http = require('http');

http.createServer((req, res) => {
  if (req.url === '/') {
    // SSR：服务端直接输出完整首屏 HTML，浏览器无需等待 JS 执行
    res.writeHead(200, { 'Content-Type': 'text/html' });
    res.end(`
      <!DOCTYPE html>
      <html>
      <head>
        <link rel='preload' href='/style.css' as='style'>
      </head>
      <body>
        <div id='app'>SSR 初始内容</div>
        <script type='module'>
          // 分包：只有当下面代码执行时，浏览器才发起对 /extra.js 的请求
          const m = await import('/extra.js');
          m.bootstrap(document.querySelector('#app'));
        </script>
      </body>
      </html>
    `);
  } else if (req.url === '/style.css') {
    // preload 命中：浏览器在解析 head 时已经提前请求该资源
    res.writeHead(200, { 'Content-Type': 'text/css' });
    res.end('body { color: #333; }');
  } else if (req.url === '/extra.js') {
    // 异步 chunk：未被 import 前不下载，实现首屏体积压缩
    res.writeHead(200, { 'Content-Type': 'text/javascript' });
    res.end(`export function bootstrap(el) {
      el.textContent += ' + dynamic';
    }`);
  }
}).listen(3000, () => {
  console.log('server on http://localhost:3000');
});
// 验证步骤：启动后访问 localhost:3000，在 DevTools Network 面板观察 style.css 在 HTML 解析早期加载，extra.js 在 import 执行后才出现。
```

### 4. 常见误区与进阶思考
1. 误区一：SSR 必然比 CSR 首屏更快。
  错误点：只看到 FCP 提前，忽略了 TTFB 可能从 20ms 涨到 200ms，也忽略了 hydration 脚本对 TBT 的影响。在弱网、高并发环境下，服务端渲染能力不足反而导致首屏响应变慢；对于静态内容，应使用 SSG/预渲染，不引入运行时渲染。

2. 误区二：preload 越多越好。
  错误点：preload 的本质是显式控制资源加载时序，而非提高网络带宽。过量 preload 会让所有预加载资源同时进入高优先级队列，挤占真正首屏关键资源（如首屏 CSS/字体）的带宽，甚至触发浏览器的加权公平队列导致关键请求被延迟。

思考题：在 HTTP/2 多路复用下，为什么将一个大 JS 文件拆成多个小 chunk 反而可能让首屏 LCP 变差？请从 TCP 拥塞控制、HTTP/2 Stream 调度、压缩字典上下文的角度解释。
