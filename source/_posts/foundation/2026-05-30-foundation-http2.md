---
title: "每日基础技术总结 · 2026-05-30 · HTTP/2 多路复用与流控制"
date: 2026-05-30 08:00:00
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-05-30 · HTTP/2 多路复用与流控制

## 📚 今日主题

> **HTTP/2 多路复用与流控制**（后端基础）

### 1. 核心概念速览
HTTP/2 多路复用（Multiplexing）是传输层上在一个 TCP 连接内同时交错传输多个二进制帧（Frame）的机制，每个帧携带所属的流标识（Stream ID）。流（Stream）是 HTTP/2 中独立的双向字节序列，用于承载一次请求-响应交换。流控制（Flow Control）则是基于滑动窗口的背压机制，用于防止发送端淹没接收端。它解决了 HTTP/1.1 的队头阻塞和连接并发受限问题，将应用层语义（请求/响应）与传输调度解耦。该机制是理解现代 Web 性能优化、负载均衡、HTTP/3 演进的基础，也是后端服务设计高并发、低延迟系统的必备认知。专业工程师必须掌握它，因为它直接影响网络栈行为、资源利用率和用户体验，且与前端常见的并发请求合并、懒加载等策略存在深层关联。

### 2. 底层原理剖析
HTTP/2 连接建立后，所有数据被封装为帧（Frame），帧有类型（HEADERS, DATA, RST_STREAM, WINDOW_UPDATE 等）和流 ID。多路复用的核心是：多个请求/响应可以交错在同一个 TCP 连接上传输，接收方按流 ID 重组消息。流控制原理：每个流和一个连接各自维护一个发送窗口（Window），初始值由 SETTINGS 帧协商（通常为 65535 字节）。发送方必须确保已发送未确认的数据量不超过窗口大小，收到 WINDOW_UPDATE 帧后窗口增大。伪代码表示发送端逻辑：
  for frame in 待发送数据:
    while frame.size > send_window[frame.stream_id]:
      wait(收到 WINDOW_UPDATE 或超时)
    发送 frame
    send_window[frame.stream_id] -= frame.size
    send_window[连接] -= frame.size
接收端每消费掉部分数据后，发送 WINDOW_UPDATE 恢复窗口。这本质上是流量控制的信用机制，与 TCP 的接收窗口类似，但 HTTP/2 在应用层实现了更细粒度（流级）的控制。对比前端已有概念：前端常理解的"并发请求数限制"是浏览器对 HTTP/1.1 连接数的人为约束；HTTP/2 多路复用将并发单位从连接降为流，浏览器不再需要限制连接数，但 TCP 层仍有拥塞控制。这类似于 Java 的接口与 TS 的接口的区别：Java 接口是编译时契约，运行时是对象方法表；TS 接口是静态类型检查，运行时不存在。HTTP/1.1 的"接口"是连接，HTTP/2 的"接口"是流，流有生命周期和状态机，连接是传输载体。

### 3. 基础代码与实战验证
以下为 Node.js 内置 http2 模块的极简验证代码（无需第三方库）：

```
// server.js
const http2 = require('http2');
const fs = require('fs');
const server = http2.createSecureServer({ key: fs.readFileSync('key.pem'), cert: fs.readFileSync('cert.pem') });
server.on('stream', (stream, headers) => {
  // 每个 stream 就是一个独立的请求流，可同时处理多个
  stream.respond({ ':status': 200 });
  stream.end(`stream ${stream.id} response`);
});
server.listen(8443);

// client.js
const http2 = require('http2');
const client = http2.connect('https://localhost:8443');
client.on('error', (err) => console.error(err));
for (let i = 0; i < 5; i++) {
  // 发起5个请求，复用同一个连接（多路复用）
  const req = client.request({ ':path': '/' });
  req.on('response', (headers) => {});
  req.on('data', (chunk) => process.stdout.write(`req${i}: ${chunk}\n`));
  req.on('end', () => {});
  req.end();
}
```

关键说明：客户端每次 `client.request()` 会在同一 TCP 连接上创建新流，底层发送 HEADERS 帧和 DATA 帧。服务端 `stream` 事件触发于每个流的 HEADERS 帧到达。流控制验证：将客户端窗口调小（通过 `localSettings`），观察发送端会等待 WINDOW_UPDATE，可用 `stream.priority` 或 `stream.pushStream` 但极简验证只需观察多个请求交错响应。

### 4. 常见误区与进阶思考
误区1：认为 HTTP/2 多路复用消除了所有队头阻塞。实际上它只消除了应用层的队头阻塞，TCP 层仍有队头阻塞——若一个 TCP 包丢失，后续所有流的数据都会被阻塞直到重传完成。HTTP/3 改用 QUIC 才解决了传输层队头阻塞。
误区2：认为流控制窗口越大越好。流控窗口决定了单流和连接的内存上界，过大会导致接收端缓冲区溢出或资源浪费，过小会限制吞吐。合理的窗口应结合 RTT 和带宽延迟积（BDP）动态调整。
思考题：若一个 HTTP/2 连接上有 100 个流，其中 1 个流发送了 2GB 的数据且不暂停，其他 99 个流是否会永远得不到带宽？为什么？请结合流级和连接级窗口、优先级和调度策略分析。
