---
title: "每日基础技术总结 · 2026-06-11 · WebSocket 握手与帧格式"
date: 2026-06-11 08:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-06-11 · WebSocket 握手与帧格式

## 📚 今日主题

> **WebSocket 握手与帧格式**（网络基础）

### 1. 核心概念速览
WebSocket 是建立在 TCP 之上的应用层全双工通信协议，通过一次 HTTP Upgrade 握手将连接从 HTTP 协议升级为 WebSocket 协议。它解决的是 HTTP 请求-响应模型下服务端无法主动推送、连接无法持久复用的问题。本质是一个长连接 + 双向消息帧的通道，消息边界由帧头显式声明，而非依赖 TCP 流的分隔符。在整个网络分层中，WebSocket 位于应用层，但不同于 HTTP 的语义，它只提供消息传输语义，不规定业务消息格式（如 JSON/二进制均可）。专业工程师必须掌握它，因为它是实时场景（IM、推送、协作、在线游戏）的基础设施，且涉及协议升级、帧解析、掩码算法、粘包处理等底层细节，任何上层封装（如 Socket.IO）都建立在对其正确理解之上。

### 2. 底层原理剖析
WebSocket 握手本质是一次带特定头部的 HTTP 请求/响应。客户端发送 GET 请求，头部包含 Upgrade: websocket、Connection: Upgrade、Sec-WebSocket-Key（随机 16 字节的 Base64 编码）。服务端校验后返回 101 Switching Protocols，并在 Sec-WebSocket-Accept 中携带对 Sec-WebSocket-Key 的证明值。计算方法为：
accept = Base64(SHA1(key + '258EAFA5-E914-47DA-95CA-C5AB0DC85B11'))。这个 GUID 是 RFC 6455 规定的固定常量，防止代理缓存和意外升级。握手成功后，TCP 连接不再承载 HTTP 语义，转而承载 WebSocket 帧。

帧格式如下（单位：bit）：
- 0: FIN，表示当前消息是否为最后一帧（分片时用）
- 1-3: RSV1-3，必须为 0，除非扩展协商
- 4-7: opcode，4 位。0x0 表示延续帧，0x1 文本，0x2 二进制，0x8 关闭，0x9 ping，0xA pong
- 8: MASK，客户端→服务端必须为 1，服务端→客户端必须为 0
- 9-15: payload len（7 位）。若值为 126，则后面紧跟 16 位无符号长度；若为 127，则后面紧跟 64 位长度（最高位必须为 0）
- 随后（如果 MASK=1）：4 字节 masking-key
- 最后：payload data。掩码算法：data[i] = encoded[i] XOR masking_key[i % 4]

帧的解析是流式的：接收端必须循环读取字节，根据长度字段判断帧边界，而不是按 TCP 数据包切分。TCP 可能粘包/拆包，所以需要缓冲区累积数据，逐字节解析帧头。分片机制中，第一个分片 opcode 为实际类型（0x1/0x2），后续分片 opcode=0x0，最后一个分片 FIN=1。控制帧（ping/pong/close）不允许分片，且 payload 长度最大 125 字节。

对比前端已有概念：HTTP 的请求-响应类似于 TS 中函数调用（调用方发起，等待返回），而 WebSocket 类似于事件发射器（EventEmitter），连接建立后双方可随时 emit 事件。更接近的对比是 XMLHttpRequest 与 WebSocket 的事件驱动差异：XHR 是一次性交互，WebSocket 是持久双向事件流。与 Java 接口和 TS 接口的差异类似——它们看似相同（都是定义协议），但 WebSocket 的握手和帧格式是真正的字节级协议，而接口只是类型层面的约定，不涉及传输层编码。理解 WebSocket 需要从字节流思考，而非从类型/对象思考。

### 3. 基础代码与实战验证
```text
以下为 Node.js 原生实现，不依赖 ws 库，直接解析握手与一帧数据。

const http = require('http');
const crypto = require('crypto');

const GUID = '258EAFA5-E914-47DA-95CA-C5AB0DC85B11';

const server = http.createServer((req, res) => {
  // 仅处理 WebSocket 握手，非 Upgrade 请求返回 400
  res.writeHead(400);
  res.end();
});

server.on('upgrade', (req, socket) => {
  const key = req.headers['sec-websocket-key'];
  if (!key) {
    socket.destroy();
    return;
  }
  // 计算 Sec-WebSocket-Accept：SHA1(key + GUID) 后 Base64
  const accept = crypto.createHash('sha1').update(key + GUID).digest('base64');

  // 响应 101，头部必须包含 Upgrade 与 Connection
  socket.write(
    'HTTP/1.1 101 Switching Protocols\r\n' +
    'Upgrade: websocket\r\n' +
    'Connection: Upgrade\r\n' +
    `Sec-WebSocket-Accept: ${accept}\r\n\r\n`
  );

  // 服务端向客户端发送一帧文本（无需掩码，MASK=0）
  // 构造帧：FIN=1, opcode=0x1, payload='hello'（长度5，小于126）
  const payload = Buffer.from('hello');
  const frame = Buffer.alloc(2 + payload.length);
  frame[0] = 0x81; // 10000001: FIN=1, opcode=0001
  frame[1] = payload.length; // MASK=0, 长度=5
  payload.copy(frame, 2);
  socket.write(frame);

  // 读取客户端发来的帧（含掩码）
  let buffer = Buffer.alloc(0);
  socket.on('data', (chunk) => {
    buffer = Buffer.concat([buffer, chunk]);
    // 解析帧头：至少需要 2 字节
    if (buffer.length < 2) return;
    const b0 = buffer[0];
    const b1 = buffer[1];
    const opcode = b0 & 0x0f;
    const masked = (b1 & 0x80) !== 0;
    let payloadLen = b1 & 0x7f;
    let offset = 2;

    if (payloadLen === 126) {
      if (buffer.length < 4) return;
      payloadLen = buffer.readUInt16BE(2);
      offset = 4;
    } else if (payloadLen === 127) {
      if (buffer.length < 10) return;
      const bigLen = buffer.readBigUInt64BE(2);
      if (bigLen > BigInt(Number.MAX_SAFE_INTEGER)) throw new Error('length too large');
      payloadLen = Number(bigLen);
      offset = 10;
    }

    // 读取掩码键（客户端→服务端必须掩码）
    let maskKey = null;
    if (masked) {
      if (buffer.length < offset + 4) return;
      maskKey = buffer.slice(offset, offset + 4);
      offset += 4;
    }

    // 等待完整负载
    if (buffer.length < offset + payloadLen) return;

    let data = buffer.slice(offset, offset + payloadLen);
    if (masked) {
      // 掩码还原：data[i] XOR maskKey[i % 4]
      for (let i = 0; i < data.length; i++) {
        data[i] ^= maskKey[i % 4];
      }
    }

    console.log('Received opcode:', opcode);
    console.log('Received payload:', data.toString('utf8'));

    // 处理完一帧后清空缓冲区（此处简化为清空，实际应保留剩余数据）
    buffer = Buffer.alloc(0);
  });
});

server.listen(8080);

注意：上述代码仅为验证握手与帧解析的最小演示。真实实现需要处理分片、控制帧、多帧缓冲、关闭流程以及大小限制。
```

### 4. 常见误区与进阶思考
常见误区一：认为 WebSocket 是基于 HTTP 的持久化消息协议。实际上握手借用 HTTP Upgrade 机制，但一旦 101 响应完成，HTTP 语义彻底消失，连接变成纯 WebSocket 帧流。很多工程师误以为可以复用 HTTP 的 keep-alive、代理、缓存等机制，导致在负载均衡、代理环境下出错。

常见误区二：忽略掩码的作用。RFC 6455 规定客户端→服务端所有帧必须掩码，服务端→客户端禁止掩码。原因在于防止早期代理缓存投毒攻击（通过可控的 payload 字节影响 HTTP 响应）。若服务端解析时不按掩码算法处理，直接使用接收到的数据，会得到乱码；若客户端未掩码，则协议不合法。

思考题：假设你收到一个 FIN=0、opcode=0x1 的分片，随后又收到一个 FIN=0、opcode=0x0 的分片，最后收到 FIN=1、opcode=0x0 的分片。请问这三帧组合后，完整的消息类型是什么？如果在第二帧和第三帧之间插入一个 opcode=0x9 的 ping 帧，接收端应如何处理？这考验对分片连续性规则和帧交错规则的理解。正确答案要求：分片消息的 opcode 由第一个分片决定，因此整体是文本消息；控制帧允许在分片之间插入，但控制帧本身不能分片，且必须在收到 FIN=1 的延续帧前处理控制帧。若违反交错规则（如在未完成分片时收到非延续帧的非控制帧），必须视为协议错误并关闭连接。
