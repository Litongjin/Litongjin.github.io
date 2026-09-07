---
title: "每日基础技术总结 · 2026-09-08 · SSE（Server-Sent Events）协议原理"
date: 2026-09-08 07:13:39
categories: [技术分享]
tags: ["技术分享", "AI 开发基础（LLM & Agent）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-08 · SSE（Server-Sent Events）协议原理

## 📚 今日主题

> **SSE（Server-Sent Events）协议原理**（AI 开发基础（LLM & Agent））

### 1. 核心概念速览
SSE（Server-Sent Events）是一种基于 HTTP 的服务端单向事件推送协议。其本质是：客户端通过普通 HTTP 请求，接受一个永不关闭的响应流（Content-Type: text/event-stream），服务端在任意时刻向该流写入符合标准格式的事件数据。它解决的问题是：在不需要客户端频繁轮询的前提下，服务端可以主动向客户端持续推送增量数据。机制上，它复用 HTTP/1.1 的分块传输编码（chunked transfer encoding），通过 keep-alive 长连接维持流式传输。在计算机体系中，SSE 位于应用层 HTTP 之上，是 Web 实时通信中轻量级的一向推送方案；在 AI 开发中，它是 LLM 流式输出的标准载体（如 OpenAI 的 stream 模式）。专业工程师必须掌握它，因为它是理解 HTTP 流、事件驱动架构以及现代 AI 应用数据管道的基础，同时它暴露了 HTTP 协议中“响应体不结束”这一底层能力的关键边界。

### 2. 底层原理剖析
1. 连接建立：客户端发起 HTTP GET 请求，请求头包含 `Accept: text/event-stream`。服务端响应头必须设置：
   - `Content-Type: text/event-stream; charset=utf-8`
   - `Cache-Control: no-cache`
   - `Connection: keep-alive`（HTTP/1.1 默认，HTTP/2 无需）
   服务端不结束响应体，而是利用分块传输编码（Transfer-Encoding: chunked）持续写入数据块。

2. 事件流协议格式：每个事件以空行 `\n` 分隔，由若干字段行组成。关键字段：
   - `data:` 事件负载，可多行，多行会被拼接为一个事件，中间以换行符连接。
   - `event:` 事件类型，默认为 `message`，客户端可通过 `addEventListener` 监听自定义类型。
   - `id:` 事件 ID，用于客户端重连时通过 `Last-Event-ID` 请求头恢复。
   - `retry:` 指定重连间隔，毫秒。
   - 以 `:` 开头的行是注释，可充当心跳包。

3. 客户端工作机理：浏览器 `EventSource` 对象维护一个持久连接，底层基于 XHR/fetch 的流式读取能力，逐步读取并解析 `text/event-stream` 字节流。解析过程是增量式的：将流入的数据按行分割，遇到空行即触发一个事件；同时根据 `id`、`retry` 管理重连逻辑。

4. 与前端已有概念的对比：
   - 与 `fetch` + `ReadableStream`：`fetch` 返回 Promise，但响应体是流式的；SSE 是在此之上封装了事件语义（自动解析、自动重连）。SSE 是协议，fetch 是通用传输。
   - 与 `WebSocket`：WebSocket 是全双工、二进制安全、独立握手（101 Switching Protocols）的协议；SSE 是单向文本流、基于 HTTP、天然支持重连。二者层级不同，不可直接替换。
   - 与前端 `EventTarget`：`EventSource` 继承 `EventTarget`，但事件源来自网络流异步分发，而非同步本地调用。
   - 类比 Java 接口与 TS 接口的区别：Java 接口是编译期类型契约，TS 接口是结构化类型系统；SSE 与 WebSocket 也是不同层次的协议，SSE 只是 HTTP 响应流的一种使用方式，而 WebSocket 是独立协议层。

### 3. 基础代码与实战验证
```text
// server.js — Node.js 原生 http 模块实现 SSE 服务端
const http = require('http');

const server = http.createServer((req, res) => {
  // 校验客户端是否接受 text/event-stream
  if (req.headers.accept && req.headers.accept.includes('text/event-stream')) {
    // 设置 SSE 必需响应头：内容类型、禁用缓存、保持连接
    res.writeHead(200, {
      'Content-Type': 'text/event-stream; charset=utf-8',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive'
    });

    // 发送一条注释作为初始化心跳，客户端忽略以冒号开头的行
    res.write(': connected\n\n');

    let count = 0;
    const timer = setInterval(() => {
      // 严格按 SSE 协议格式写事件：每个事件以空行分隔
      res.write(`id: ${count}\n`);       // 设置事件 ID，断线重连时携带 Last-Event-ID
      res.write(`event: message\n`);     // 自定义事件类型，默认即 message，可省略
      res.write(`data: ${JSON.stringify({ time: Date.now(), count })}\n\n`); // data 字段 + 空行表示事件结束
      count++;
    }, 1000);

    // 当客户端断开连接时，清理定时器并结束响应流
    req.on('close', () => {
      clearInterval(timer);
      res.end();
    });
  } else {
    res.writeHead(404);
    res.end();
  }
});

server.listen(3000, () => console.log('SSE server at http://localhost:3000'));

// client.js — 浏览器环境运行
const es = new EventSource('http://localhost:3000');

// 监听 message 类型事件（默认类型）
es.addEventListener('message', (event) => {
  console.log('收到事件:', event.data);
  // event.data 是 SSE 协议中 data: 字段拼接后的字符串
  // 底层已经完成流式解析，这里拿到的是完整消息
});

es.onerror = (e) => {
  console.error('连接异常，自动重连中...', e);
  // EventSource 会自动重连，若服务器设置了 id，则重连请求头会包含 Last-Event-ID
};
```

### 4. 常见误区与进阶思考
1. 误区一：将 SSE 视为 WebSocket 的弱化版。实际上 SSE 在“服务端→客户端单向推送”场景下是更优选择：它基于 HTTP，无需协议升级；内置自动重连和事件 ID；开销远低于 WebSocket。WebSocket 的强项是双向、低延迟的实时交互，但复杂度更高且不提供自动重连。对于 LLM token 流式输出这类单向推送，SSE 是标准且更匹配的。

2. 误区二：认为通过 `res.write()` 随便写内容就是发送 SSE 事件。底层 `res.write()` 只是向 TCP 缓冲区写入原始字节；`EventSource` 必须依据 `data`/`event`/`id`/`retry` 字段以及空行分隔符来解析。若格式不规范（例如缺少结尾的空行，或 data 行之间混入其他字段），会导致事件无法触发、数据被拼接错乱，甚至连接挂起。必须严格遵循 `字段: 值` 和 `\n\n` 边界规则。

3. 进阶思考题：当服务端按以下顺序写入字节流：`data: A\n`、`data: B\n\n`，客户端 `EventSource` 最后会收到一个什么消息？其内容是什么？请解释 SSE 中“多行 data 字段拼接”的规则与事件边界（空行）的判定逻辑，并说明在什么场景下必须使用 `id` 和 `retry` 字段来保证可靠投递。

（思考题答案：客户端会收到一个类型为 `message`、`data` 值为 `'A\nB'` 的事件。因为连续多个 `data:` 行会被拼接成一个事件，拼接符是换行符；空行标识事件结束。若网络中断，`EventSource` 会自动重连，并通过 `Last-Event-ID` 请求头把最后收到的 `id` 发送给服务端；服务端可据此从断点后续发。`retry` 则控制重连等待时长，避免服务端雪崩。）
