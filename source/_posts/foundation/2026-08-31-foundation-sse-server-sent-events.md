---
title: "每日基础技术总结 · 2026-08-31 · SSE（Server-Sent Events）协议原理"
date: 2026-08-31 06:55:51
categories: [技术分享]
tags: ["技术分享", "AI 开发基础（LLM & Agent）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-31 · SSE（Server-Sent Events）协议原理

## 📚 今日主题

> **SSE（Server-Sent Events）协议原理**（AI 开发基础（LLM & Agent））

### 1. 核心概念速览
SSE（Server-Sent Events）是一种基于 HTTP 的服务端单向推送协议，标准媒体类型为 text/event-stream。其本质是：客户端发起一个普通 GET 请求，服务端保持该响应连接不关闭，以 UTF-8 编码按事件帧格式持续写回数据；浏览器端通过 EventSource API 消费。它解决的问题是：在不需要双向通信的场景下，以最低成本实现服务器到客户端的持续流式传输，并原生具备断线重连、事件 ID 续传的能力。机制上没有任何新协议，只依赖 HTTP 的 chunked 传输和长连接。在计算机体系中的位置：处于 HTTP 应用层之上、传输层 TCP 之上，属于 Web 实时通信协议谱系中与 WebSocket 并列但更轻量的方案。在 AI 开发中，OpenAI 兼容的 chat/completions 接口默认输出就是 SSE，LLM token 逐字流式返回均走该协议；专业工程师必须掌握它，才能正确接入模型网关、处理流式响应、实现取消、心跳与重连等生产级细节。

### 2. 底层原理剖析
客户端构造 EventSource(url) 时发送 HTTP GET，请求头 Accept: text/event-stream。服务端响应头必须包含 Content-Type: text/event-stream; charset=utf-8，并禁用缓存；HTTP/1.1 下通过 Transfer-Encoding: chunked 分块写入。连接建立后，服务端写入遵循 event-stream 格式的文本块，每条事件由若干字段行组成，字段行格式为 field: value，字段包括：
data: 数据内容，可多行，每行都是 data:，客户端会合并为一条，用换行符连接。
event: 自定义事件类型，缺省为 message。
id: 事件 ID，浏览器会记录最后一次收到的 id，并在重连时放入 Last-Event-ID 请求头。
retry: 重连间隔毫秒数。
事件以空行 \n\n 结尾。客户端收到完整空行后按事件分发给对应 listener。服务端发送 : comment 行可作注释/心跳保持连接。

SSE 与前端已有概念的对比：
- 与 WebSocket：WebSocket 是全双工二进制长连接，需要独立的 ws:// 升级握手；SSE 是单向文本推送，直接复用普通 HTTP 请求，浏览器自动处理重连和 Last-Event-ID，恢复成本低；LLM 场景服务端流式输出且客户端不需要上行，SSE 更合适。
- 与 fetch + ReadableStream：fetch 流式读取只是拿到原始 chunk，没有协议层面的事件边界，需要自己解析；EventSource 在浏览器内部完成了事件帧解析、事件派发、自动重连。SSE 协议本身就是一种“服务端事件流”的帧格式，不是简单的 TCP 流。

本质流程：
1. 浏览器发起 GET。
2. 服务端写响应头，flush。
3. 服务端循环写入 id: n\nevent: message\ndata: {...}\n\n，每写一次即一个事件帧。
4. 连接断开时，浏览器用 Last-Event-ID 发起新连接，服务端可根据该值从断点继续发送。

### 3. 基础代码与实战验证
```text
Node.js 服务端（server.js）：

const http = require('http');

http.createServer((req, res) => {
  if (req.url !== '/events') { res.end('not found'); return; }

  // 设置必要的响应头：类型必须是 text/event-stream，禁止缓存
  res.writeHead(200, {
    'Content-Type': 'text/event-stream; charset=utf-8',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive'
  });
  res.flushHeaders(); // 立刻将响应头发送出去，使客户端能进入等待流状态

  let seq = 0;

  // 每 1 秒推送一个事件帧。底层是 res.write 将字符串切成 chunk 写入 TCP socket
  const timer = setInterval(() => {
    seq++;
    const payload = JSON.stringify({ token: `token-${seq}`, seq });
    // 事件帧格式：id: 序列号，event: 事件名，data: JSON 数据，空行结束
    res.write(`id: ${seq}\n`);
    res.write(`event: token\n`);
    res.write(`data: ${payload}\n`);
    res.write(`\n`);
  }, 1000);

  req.on('close', () => clearInterval(timer)); // 客户端断连时清理资源
}).listen(3000);

浏览器客户端：

const es = new EventSource('http://localhost:3000/events');

// 收到 event: token 的事件帧时触发
es.addEventListener('token', (event) => {
  const data = JSON.parse(event.data);
  console.log(data);
});

// 默认 message 事件；同时连接错误和自动重连由浏览器处理
es.onerror = () => console.log('连接异常，浏览器会自动重连');
```

### 4. 常见误区与进阶思考
误区1：把 SSE 当 WebSocket 使用。SSE 是单向的，服务器无法通过同一条连接主动收消息；若客户端需要上行数据，应选择 WebSocket。否则在为不需要双向的流式场景强行引入 WebSocket，徒增握手和编解码复杂度。另一变体是误以为心跳就是重连机制，其实 retry 与 id 已覆盖自动重连，心跳只是防止中间代理/网关静默断开连接的补充。

误区2：忽略代理/网关的缓冲。nginx、CDN、云网关默认会缓冲 HTTP 响应，导致客户端迟迟收不到 token。需要显式关闭缓冲，例如在 nginx 上设置 X-Accel-Buffering: no；同时应避免对 text/event-stream 使用 gzip 压缩，因为压缩会引入额外缓冲和延迟。

思考题：如果服务端在发送 id: 10 之后立刻断开，客户端重连时请求头 Last-Event-ID 的值是多少？服务端应如何根据此 ID 恢复发送，才能保证事件不重不丢？请画出事件帧写入和断连恢复的时间线。
