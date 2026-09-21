---
title: "每日基础技术总结 · 2025-04-12 · WebSocket 全双工通信与握手升级"
date: 2025-04-12 20:00:00
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础（扩展）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2025-04-12 · WebSocket 全双工通信与握手升级

## 📚 今日主题

> **WebSocket 全双工通信与握手升级**（前端底层与计算机基础（扩展））

### 1. 核心概念速览
WebSocket 是一种在单个 TCP 连接上进行全双工通信的协议，旨在解决 HTTP 长轮询（Long Polling）在高并发场景下的高延迟与资源浪费问题。其本质是通过一次性的 HTTP 握手请求，将连接从单向的‘请求-响应’模式升级为双向的‘消息流’模式，后续数据帧不再遵循 HTTP 语义头，而是采用自定义的二进制/文本帧格式。在计算机体系结构中，它位于应用层，依赖于传输层的 TCP；在前端工程中，它是实现实时交互（如聊天、协同编辑、即时行情推送）的标准基础设施，也是现代 WebRTC 和 WebSocket Server（如 Node.js ws）的核心通信基石，专业工程师必须掌握以理解状态机转换、二进制帧结构及连接生命周期管理。

### 2. 底层原理剖析
1. 握手升级机制：客户端发送包含特定 Header（Upgrade: websocket, Connection: Upgrade, Sec-WebSocket-Key）的 HTTP GET 请求；服务端验证 Key（基于 RFC 6455 的 SHA-1 哈希运算），返回 101 Switching Protocols 状态码并附带 Sec-WebSocket-Accept 响应。此后 TCP 连接复用，但 HTTP 解析器停止工作。
2. 帧结构解析：协议定义了 Fin、Opcode、Mask、Payload length 等字段。Fin 表示是否最后一个分片；Opcode 定义类型（文本 0x1，二进制 0x2，关闭 0x8，心跳 Ping 0x9/Pong 0xA）；Mask 位用于防止中间缓存污染（仅客户端发往服务端的数据必须掩码）。
3. 对比前端概念：不同于 TypeScript Interface（编译时静态类型检查，不产生运行时行为），WebSocket 是运行时的动态通信契约；不同于 Java 接口（抽象方法集合），WebSocket 是一套严格的二进制帧状态机。前端已有的 AJAX/Fetch 是基于无状态、请求驱动的单工模型，而 WebSocket 是基于有状态、事件驱动的全双工模型，前者关注数据传输的效率（压缩、合并），后者关注连接状态的维护（重连、保活、乱序重组）。

### 3. 基础代码与实战验证
```text
// 使用 Node.js net.Socket 模拟原生 WebSocket 握手与数据读取
const net = require('net');
const crypto = require('crypto');

// 1. 建立 TCP 连接
const server = net.createServer((socket) => {
  let buffer = '';
  // 监听原始数据，因为尚未解析为 WS 帧
  socket.on('data', (chunk) => {
    buffer += chunk.toString();
    if (buffer.includes('\r\n\r\n')) {
      // 检测到头部结束，处理握手
      const headers = buffer.split('\r\n\r\n')[0];
      const keyMatch = headers.match(/Sec-WebSocket-Key: (.+)/);
      if (keyMatch && headers.includes('Upgrade: websocket')) {
        const key = keyMatch[1].trim();
        // 2. 计算 Sec-WebSocket-Accept
        const acceptKey = crypto.createHash('sha1')
          .update(key + '258EAFA5-E914-47DA-95CA-C5AB0DC85B11') // GUID 固定值
          .digest('base64');
        
        // 3. 发送 101 Switching Protocols
        const response = [
          'HTTP/1.1 101 Switching Protocols',
          'Upgrade: websocket',
          'Connection: Upgrade',
          `Sec-WebSocket-Accept: ${acceptKey}`,
          '',
          ''
        ].join('\r\n');
        socket.write(response);
        console.log('Handshake complete, switching to full-duplex mode.');
      }
    }
  });
});
server.listen(8765);
```

### 4. 常见误区与进阶思考
误区一：认为 WebSocket 是 HTTP 的一部分或只是 HTTP 的一个属性。实际上，一旦握手完成，后续的数据交换完全脱离 HTTP 协议栈，HTTP 服务器无法直接处理后续的二进制帧，必须使用专门的 WS 解析库或自行实现帧解包逻辑。
误区二：忽视 TCP 粘包与分片问题。WebSocket 帧天然支持大数据切分（Fragmentation），但底层 TCP 流仍是字节流，应用层需根据 Fin 位和 Opcode 重组完整消息，不能简单地将 Socket 接收到的 Buffer 当作一条完整业务消息。
思考题：在 WebSocket 握手完成后，若此时出现网络波动导致 TCP 半关闭（Half-Open），上层应用如何感知？请结合 TCP KeepAlive 机制与 WebSocket 的 Ping/Pong opcode 设计，分析两者在保持连接活性上的层级差异及其潜在的死锁风险。
