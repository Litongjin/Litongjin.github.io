---
title: "每日基础技术总结 · 2026-08-21 · SSE（Server-Sent Events）协议原理"
date: 2026-08-21 17:41:49
categories: [技术分享]
tags: ["技术分享", "AI 开发基础（LLM & Agent）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-21 · SSE（Server-Sent Events）协议原理

## 📚 今日主题

> **SSE（Server-Sent Events）协议原理**（AI 开发基础（LLM & Agent））

### 1. 核心概念速览
SSE（Server-Sent Events）是一种基于 HTTP 的服务器向客户端单向推送事件的协议。其本质是客户端通过 HTTP 请求建立一条长连接，服务器在该连接上以纯文本流（text/event-stream）的形式分块写入数据，客户端通过 EventSource API 按事件类型接收并处理。它解决的核心问题是：在不需要双向通信的前提下，如何以标准 HTTP 协议实现服务器到客户端的持续、可靠、自动重连的数据推送。与 WebSocket 不同，SSE 只支持服务器到客户端，但天然具备 HTTP 的兼容性、自动重连、事件 ID 和自定义事件类型等机制。在 AI 开发中，SSE 是 LLM 流式输出（如 OpenAI 兼容接口的 /chat/completions 流式响应）的标准传输方式，用于将 token 逐个或分批推送给前端。专业工程师必须掌握它，因为它是构建流式 AI 应用（聊天机器人、实时生成）的底层协议，且其语义与前端熟悉的 XMLHttpRequest/fetch 完全不同，误解其传输模型会导致流式处理失败。

### 2. 底层原理剖析
SSE 的底层机制是 HTTP 长连接的流式响应。当客户端发起 GET 请求并设置 Accept: text/event-stream 后，服务器返回 200 状态码，并设置 Content-Type: text/event-stream，同时关闭对响应的缓冲（如 Transfer-Encoding: chunked 或直接 flush）。此后服务器可以持续向该响应体写入文本块，每个事件由多个字段组成，字段之间以换行符分隔，事件之间以空行（两个连续换行）分隔。基本事件格式如下：

id: <事件ID>\ndata: <第一行数据>\ndata: <第二行数据>\nevent: <事件类型>\nretry: <重连毫秒数>\n\n

其中 data 字段可以出现多行，客户端会将连续多行 data 按换行拼接成一个完整数据；event 字段指定事件名，若省略则默认触发 message 事件；id 字段用于断线重连时发送 Last-Event-ID 请求头；retry 字段告诉客户端重连间隔。服务器发送注释行（以冒号开头）可作为心跳包，防止代理超时。

传输过程中，响应体不是一次返回，而是持续打开，HTTP 1.1 的 chunked 编码允许服务器不断发送数据块。与前端已有的 fetch 相比，fetch 的 response.body 是一个 ReadableStream，SSE 本质就是对该流按特定文本协议进行解析。但 SSE 不是对 ReadableStream 的简单封装，因为 EventSource 内置了重连、事件 ID、自定义事件等状态管理机制。与 WebSocket 相比，WebSocket 是双向二进制/文本帧协议，需要升级握手，而 SSE 仍然是普通 HTTP 响应，这意味着它天然穿越防火墙和代理，且可以使用普通 HTTP 缓存、认证、重定向等机制。

底层状态机：客户端维护一个 readyState（CONNECTING/OPEN/CLOSED），连接断开后自动根据 retry 字段或默认值（3 秒）重试，若最后一次收到的事件带有 id 字段，重连请求头会自动带上 Last-Event-ID。服务器无法主动关闭连接（除非返回非 200 或非 text/event-stream 的响应），客户端调用 EventSource.close() 才能关闭。

伪代码描述：

客户端：
1. 构造 HTTP GET 请求，头 Accept: text/event-stream，若之前有 lastEventId，则加 Last-Event-ID
2. 收到响应头，检查 Content-Type，若不匹配则触发 error 并关闭
3. 读取响应体字节流，按 UTF-8 解码，按 '\n' 缓冲行，按空行分割事件块
4. 解析事件块中的字段（id, event, data, retry），生成 Event 对象派发
5. 若流断开，检查请求头中 retry 字段设置延迟，然后回到步骤 1（自动重连）

服务器：
1. 收到请求，校验 Accept 头
2. 设置 Content-Type: text/event-stream; charset=utf-8，Cache-Control: no-cache，Connection: keep-alive
3. 循环写入事件块（每个事件以\n\n结尾），每次写入后 flush 底层 socket
4. 需要保持连接时周期发送冒号注释行作为心跳
5. 当服务器异常终止时，直接关闭响应体，客户端会检测到断流并重连

### 3. 基础代码与实战验证
```text
// 纯前端验证 SSE 协议，不依赖任何框架。底层网络由浏览器 EventSource 实现，但协议解析和重连机制需要理解。
// 由于 EventSource 是浏览器内置 API，无法在 Node 中直接使用，这里展示服务端和客户端的最小验证代码。

// ---- 服务端（Node.js 原生 http 模块）----
const http = require('http');

http.createServer((req, res) => {
  // 关键：只有 Accept 头包含 text/event-stream 的请求才走 SSE 逻辑
  if (req.url === '/events' && req.headers.accept === 'text/event-stream') {
    res.writeHead(200, {
      'Content-Type': 'text/event-stream; charset=utf-8', // 告知客户端这是 SSE 流
      'Cache-Control': 'no-cache', // 禁用缓存，确保每次都是新数据
      'Connection': 'keep-alive', // 保持 TCP 连接不关闭
      'Access-Control-Allow-Origin': '*' // 跨域支持
    });

    let eventId = 0;
    // 每 1 秒推送一个事件，验证分块传输
    const timer = setInterval(() => {
      eventId++;
      // SSE 协议格式：id 行 + data 行 + 空行结束事件
      // 必须写入空行（\n\n），客户端才会解析为一个完整事件
      res.write(`id: ${eventId}\n`);
      res.write(`data: {"message": "hello", "id": ${eventId}}\n`);
      res.write(`\n`);
      // 注意：res.write 不会自动 flush，但对于 Node http 响应，数据会写入内核缓冲；高并发下可调用 res.flushHeaders() 或使用压缩等机制，此处不深入
    }, 1000);

    req.on('close', () => { // 客户端断开时清除定时器
      clearInterval(timer);
      res.end();
    });
  } else {
    res.writeHead(404);
    res.end();
  }
}).listen(3000);

// ---- 客户端（浏览器原生 EventSource）----
const es = new EventSource('http://localhost:3000/events');

// message 事件：当事件块中没有 event 字段时默认触发
es.onmessage = (event) => {
  console.log('收到数据:', event.data); // event.data 为字符串
};

// 监听自定义事件：若服务器写入 event: custom\n，则触发 custom 事件
// es.addEventListener('custom', (e) => { ... });

es.onerror = (e) => {
  // 连接错误或断线时触发。EventSource 会自动重连，无需手动处理
  console.error('SSE 错误:', e);
  // 若希望手动关闭：es.close();
};

// 上述代码中，EventSource 底层会自动完成：
// 1. 发送 GET 请求，Accept: text/event-stream
// 2. 解析流式响应，按空行切分事件
// 3. 处理重连逻辑和 Last-Event-ID
// 开发者只需关注事件回调，这验证了 SSE 协议的核心机制。
```

### 4. 常见误区与进阶思考
误区一：认为 SSE 可以像 WebSocket 一样从客户端向服务器发送数据。实际上 SSE 是单工协议，客户端无法在同一个连接中发送额外数据。若需要发送数据，只能额外发起 HTTP 请求（如 fetch）。这个误区会导致设计上试图在 EventSource 上调用 send 方法，或者期望服务器能通过 req 对象读取后续数据，但 SSE 的请求体在握手后就不再被使用，服务器只能从 req 上读取初始请求信息（如查询参数、Header）。

误区二：认为 SSE 的数据是普通 JSON 一次性返回，或者认为响应体的 content-type 为 application/json。SSE 是纯文本流，且必须是 text/event-stream。如果将服务器返回的流当作 JSON 解析，会得到语法错误。另一个常见错误是忘记在服务端关闭缓冲（如 nginx 的 proxy_buffering），导致数据不是逐块到达而是攒到一定大小才一次性 flush，造成前端长时间无响应。

深度思考题：在一个 SSE 连接中，服务器连续发送了两个事件，第一个事件块带有 id: 1，第二个事件块没有 id 字段。当连接在第二个事件发送后断开，客户端自动重连时，请求头 Last-Event-ID 的值是多少？为什么？请结合 SSE 协议的字段继承规则与重连机制，解释你对底层状态的理解。
