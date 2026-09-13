---
title: "每日基础技术总结 · 2026-09-14 · SSE（Server-Sent Events）协议原理"
date: 2026-09-14 07:02:36
categories: [技术分享]
tags: ["技术分享", "AI 开发基础（LLM & Agent）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-14 · SSE（Server-Sent Events）协议原理

## 📚 今日主题

> **SSE（Server-Sent Events）协议原理**（AI 开发基础（LLM & Agent））

### 1. 核心概念速览
SSE（Server-Sent Events）是建立在 HTTP 协议之上的服务端单向推送机制，由 W3C 的 EventSource API 标准化。其本质是：客户端通过普通 HTTP GET 请求建立一个长连接，服务端以 `text/event-stream` 的 MIME 类型分块（chunked）持续返回 UTF-8 编码的文本数据，每个数据块遵循 `data:`、`event:`、`id:`、`retry:` 等字段组成的事件流格式。SSE 解决的核心问题是：在无需客户端反复轮询（polling）的前提下，服务端能够主动向浏览器或任意 HTTP 客户端异步推送结构化事件。与 WebSocket 不同，SSE 是单工（服务端到客户端）的、基于文本的、天然具备 HTTP 语义（如认证、重定向、代理）的协议；它利用 HTTP/1.1 的 chunked transfer encoding 或 HTTP/2 的流（stream）机制实现流式传输。在整个计算机体系中的位置：它属于应用层协议 HTTP 的扩展用法，位于 OSI 七层模型的应用层；在 AI 开发中，它是 LLM（如 OpenAI）流式输出 token 的标准接入方式之一（另一种是 WebSocket 或纯轮询）。专业工程师必须掌握它，因为它是实现实时通知、日志流、命令行输出、模型推理 token 流等场景的最轻量、最符合 REST 生态的机制，且其底层涉及 HTTP 连接生命周期、编码、缓冲、断线重连等核心网络细节，理解它能打通前端事件驱动模型与后端流式响应的本质。

### 2. 底层原理剖析
底层运行机制分五个层次：
1. HTTP 长连接：客户端发起 GET 请求，请求头包含 `Accept: text/event-stream`（可选，但规范要求）。服务端收到后，不结束响应，而是持续往响应体写入数据；HTTP/1.1 下利用 `Transfer-Encoding: chunked`，每个 chunk 即一段事件数据；HTTP/2 下则为独立流，可复用连接。
2. 事件流格式：每个事件由若干字段行（field: value）和一个空行组成。字段包括：`data:`（数据内容，可多行，拼接时用换行符；多行 data 在客户端会被合并为一个事件）；`event:`（事件类型，默认 `message`，对应 EventSource 的 `addEventListener` 类型）；`id:`（事件 ID，用于 Last-Event-ID 重连）；`retry:`（重连间隔毫秒）。注释行以 `:` 开头，用于心跳保活。
3. 数据传输与缓冲：服务端必须显式 flush（刷新）输出缓冲，否则数据会积压在操作系统或代理层。每个事件块写入后，客户端 EventSource 会立即触发对应的 DOM 事件。
4. 断线重连机制：当连接意外断开，浏览器自动重新发起请求；如果上次响应中带有 `id:`，则新请求的请求头会自动携带 `Last-Event-ID`；服务端可据此从断点继续推送，实现可靠的事件流（at-least-once 语义，因为客户端可能重复收到已处理事件）。服务端也可通过关闭连接来主动重置重连。
5. 与前端已有概念的对比：SSE 之于 HTTP，类似于 `ReadableStream` 之于 `ArrayBuffer`——前者是持续数据流，后者是完整数据块。EventSource API 类似 `EventTarget`，事件分发机制与浏览器 DOM 事件一致，但底层是文本流解析器。可以类比：服务端 send 事件相当于前端 `dispatchEvent`，而 `data:` 字段相当于 `event.detail`。注意：SSE 不是流式传输新协议，而是对 HTTP 响应体的持续利用，因此它受 HTTP 代理缓冲、连接超时等影响。实现时，服务端应禁用缓冲（如 Node.js 中 `res.flushHeaders()` 和 `res.write()` 后不调 `res.end()`），并使用心跳注释（`retry` 或定时发送 `: ping`）保活。
伪代码流程（服务端）：
```
handle_http_request(req, res):
  if req.method == GET and req.accept == 'text/event-stream':
    res.statusCode = 200
    res.setHeader('Content-Type', 'text/event-stream')
    res.setHeader('Cache-Control', 'no-cache')
    res.setHeader('Connection', 'keep-alive') // HTTP/1.1 默认
    res.flushHeaders() // 立即发送响应头，开启长连接
    loop:
      event = generate_event()
      res.write('id: ' + event.id + '\n')   // 可选
      res.write('event: ' + event.type + '\n') // 可选，默认 message
      res.write('data: ' + event.data + '\n')  // 可多行
      res.write('\n')  // 空行分隔
      res.flush() // 关键：强制发送到客户端，否则被缓冲
      sleep(interval)
```
客户端 EventSource 内部解析此文本流，按空行切分事件，并将字段映射到事件对象。

### 3. 基础代码与实战验证
```text
// 极简实现：Node.js 纯原生 http 模块作为服务端，浏览器原生 EventSource 客户端。
// 服务端代码 server.js
const http = require('http');

http.createServer((req, res) => {
  // 只处理 SSE 请求，其他返回 404
  if (req.url !== '/events') {
    res.writeHead(404);
    res.end();
    return;
  }

  // 关键响应头：必须为 text/event-stream
  res.writeHead(200, {
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive', // HTTP/1.1 默认，但显式声明更明确
  });

  // 立即 flush 响应头，使客户端能建立 EventSource 连接并等待数据
  res.flushHeaders();

  let counter = 0;
  // 每 1 秒发送一个事件，验证流式推送与自动重连
  const timer = setInterval(() => {
    counter++;
    // 按 SSE 协议格式化：data: + 空格 + 数据 + 换行，然后空行结束事件
    // 此处不使用 id 字段，若需要断点续传则添加 `id: ${counter}\n`
    res.write(`data: {"count": ${counter}, "timestamp": ${Date.now()}}\n`);
    res.write('\n');
    // 显式刷新（Node.js 中 res.write 通常直接发到 socket，但某些代理/环境需要 flush）
    // 实际在 Node 中无需额外调用，但此处强调底层缓冲机制
  }, 1000);

  // 客户端断开时清理定时器，防止资源泄漏
  req.on('close', () => {
    clearInterval(timer);
    res.end();
  });
}).listen(3000);

// 浏览器客户端（HTML 内嵌）
// 使用原生 EventSource，无需任何第三方库
const source = new EventSource('/events');

// 默认事件类型为 'message'，对应 data: 字段的内容（不含 data: 前缀和末尾换行）
source.onmessage = (e) => {
  // e.data 是字符串，需要 JSON.parse 获取对象
  console.log('收到事件:', e.data);
};

// 可选：监听自定义事件类型，需服务端发送 `event: custom\n`
// source.addEventListener('custom', (e) => { ... });

// 错误处理：自动重连由浏览器内部实现，无需手动处理
source.onerror = (e) => {
  console.error('连接异常，将自动重连（默认重试时间 3s）', e);
};

// 关闭连接：source.close();

// 服务端运行：node server.js 后，打开浏览器访问静态 HTML，即可看到每秒输出。
// 关键底层运作：
// 1. res.flushHeaders() 立刻返回 HTTP 200，浏览器 EventSource 才能识别成功；
// 2. 每次 res.write() 追加字节到响应体，浏览器端增量解析；
// 3. 事件间的空行是事件分隔符，客户端据此触发 message 事件；
// 4. 若连接断开，浏览器自动发送新请求，服务端重新进入流程。
```

### 4. 常见误区与进阶思考
误区一：认为 SSE 与 WebSocket 一样是全双工。实际上 SSE 是单工——客户端只能通过单独的普通 HTTP 请求（非 SSE 连接）发送数据，SSE 连接本身只允许服务端写。若需要双向实时通信，必须另建请求或使用 WebSocket。底层原因在于 SSE 是单向响应体的流式扩展，无法在同一个 HTTP 响应通道中承载上行数据。
误区二：忽视代理服务器与中间层缓冲。很多工程师本地直连成功，但部署到 Nginx 等反向代理后 SSE 无响应。原因是代理默认缓冲整个响应体，只有等到连接关闭才转发。必须对 SSE 路径设置 `proxy_buffering off;` 以及合适的 `proxy_read_timeout`。同样，HTTP/1.1 的 chunked 编码需要服务端显式 flush，某些框架（如默认 no-buffer 需配置）会在事件数据积压 4KB 或达到某个时间阈值才发送，导致客户端收到突发批量数据而非流式。
思考题：如果服务端在某次发送的事件中设置了 `id: 100`，随后在客户端处理完该事件后网络断开，客户端自动重连时发送的 `Last-Event-ID` 是多少？服务端接收到该头部后应如何决定从哪条事件继续发送？如果客户端已经处理到了事件 id=100，而服务端从 id=101 开始发送，是否可能丢事件？请结合 SSE 的 at-least-once 语义与 HTTP 无状态特性，分析是否需要在上层引入去重机制。
