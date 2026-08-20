---
title: "每日基础技术总结 · 2026-08-13 · WebSocket 的关闭握手与心跳保活"
date: 2026-08-13 08:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-08-13 · WebSocket 的关闭握手与心跳保活

## 📚 今日主题

> **WebSocket 的关闭握手与心跳保活**（网络基础）

### 1. 核心概念速览
WebSocket 关闭握手（Closing Handshake）是协议层定义的优雅终止连接机制，通过交换 Close 控制帧（opcode=0x8）协商状态码和原因，确保双方在关闭 TCP 连接前有序释放资源，避免数据丢失。心跳保活（Heartbeat）基于 Ping/Pong 控制帧（opcode=0x9/0xA），由一端发送 Ping，对端必须回 Pong，用于检测连接活性、维持中间设备（NAT/代理）的映射、识别半开连接。本质上是应用层协议对连接生命周期的显式管理，与 TCP 的 FIN/ACK 四路挥手不同，WebSocket 的关闭握手是带语义的应用层协商，而心跳是控制帧在数据通道中的带内信令。在整个体系中，WebSocket 位于 TCP 之上，提供全双工、消息边界的应用层协议，掌握关闭与心跳是构建可靠实时系统（推送、聊天、游戏）的基础，也是排查连接异常、设计重连策略的前提。

### 2. 底层原理剖析
关闭握手：连接处于 OPEN 状态时，任一端可发起关闭。发送 Close 帧后进入 CLOSING 状态。对端收到 Close 帧后，必须回复一个 Close 帧（通常回显相同状态码），然后主动关闭 TCP 连接（发送 FIN）。发起方收到对端的 Close 帧后，关闭 TCP。若双方同时发送 Close，则各自收到对端 Close 后均关闭。若一方长期未收到回复，可强制销毁 socket。Close 帧负载包含 2 字节状态码和 UTF-8 原因。状态码 1000 表示正常，1001 表示 Going Away，1002 协议错误等。注意：若收到非法的状态码或负载长度小于 2，应回复 1002。关闭后连接不可复用。

心跳保活：Ping 帧可携带应用数据（最大 125 字节），对端收到后必须回 Pong 帧且负载必须与 Ping 完全一致（用于匹配）。Ping/Pong 是控制帧，FIN 必须为 1，不允许分片。它们可以随时插入数据流，甚至可以在分片消息之间。心跳机制不是协议强制要求，但由库或应用实现。定时发送 Ping，若在超时时间内未收到 Pong，则判定连接不可用，主动关闭。这与 TCP keep-alive 不同：TCP keep-alive 由操作系统维护，默认间隔 2 小时，且不携带数据；HTTP keep-alive 仅指 TCP 连接复用，没有应用层确认。前端对比：类似 EventSource 的自动重连，但 EventSource 基于 HTTP 长连接，无法发送上行消息；WebSocket 心跳是双向的，且是帧级别的确认。

### 3. 基础代码与实战验证
```text
// 最小 WebSocket 帧构造（服务端→客户端，无掩码）
function encodeFrame(opcode, payload) {
  const len = payload.length;
  let header;
  if (len < 126) {
    header = Buffer.alloc(2);
    header[1] = len;
  } else if (len < 65536) {
    header = Buffer.alloc(4);
    header[1] = 126;
    header.writeUInt16BE(len, 2);
  } else {
    header = Buffer.alloc(10);
    header[1] = 127;
    header.writeBigUInt64BE(BigInt(len), 2);
  }
  header[0] = 0x80 | opcode; // FIN=1 + opcode
  return Buffer.concat([header, payload]);
}

// 解析帧（单帧，实际需处理粘包/半包）
function decodeFrame(buffer) {
  const opcode = buffer[0] & 0x0f;
  const masked = (buffer[1] & 0x80) !== 0;
  let len = buffer[1] & 0x7f;
  let offset = 2;
  if (len === 126) {
    len = buffer.readUInt16BE(offset);
    offset += 2;
  } else if (len === 127) {
    len = Number(buffer.readBigUInt64BE(offset));
    offset += 8;
  }
  const mask = masked ? buffer.slice(offset, offset + 4) : null;
  if (mask) offset += 4;
  let payload = buffer.slice(offset, offset + len);
  if (mask) {
    payload = Buffer.from(payload.map((byte, i) => byte ^ mask[i % 4]));
  }
  return { opcode, payload };
}

// 处理帧：Close/Ping/Pong
function handleFrame(frame, socket) {
  if (frame.opcode === 0x8) { // Close
    // 解析状态码（前2字节），无则默认1000
    const code = frame.payload.length >= 2 ? frame.payload.readUInt16BE(0) : 1000;
    // 回同一个状态码的 Close 帧
    const reply = Buffer.alloc(4);
    reply[0] = 0x88;
    reply[1] = 2;
    reply.writeUInt16BE(code, 2);
    socket.write(reply);
    // 发送 FIN 关闭 TCP
    socket.end();
  } else if (frame.opcode === 0x9) { // Ping
    // 必须回 Pong，负载相同
    const pong = Buffer.alloc(2 + frame.payload.length);
    pong[0] = 0x8A;
    pong[1] = frame.payload.length;
    frame.payload.copy(pong, 2);
    socket.write(pong);
  }
}

// 心跳：每30s发Ping，10s未收到Pong则销毁
socket.setTimeout(10000, () => socket.destroy());
const timer = setInterval(() => {
  if (!socket.destroyed) {
    const ping = Buffer.alloc(4);
    ping[0] = 0x89;
    ping[1] = 2;
    ping[2] = 0x68;
    ping[3] = 0x62;
    socket.write(ping);
  } else {
    clearInterval(timer);
  }
}, 30000);
```

### 4. 常见误区与进阶思考
误区1：认为关闭握手必须双方都主动发送 Close 帧才算优雅。实际上，任一端发送 Close 后，对端必须回 Close，然后发起方收到即完成，并不要求对端也主动发起。若对端不应答，发起方可以强制关闭，这种强制关闭与 TCP RST 类似。

误区2：认为心跳只要定时发 Ping 就足够了，不检查 Pong。实际上，Ping 只是探测，活性确认依赖于对端在协议层面必须回复的 Pong。如果只发 Ping 而不检查超时，无法发现对端已死或网络中断。另外，Ping/Pong 是控制帧，不会触发 onmessage，必须通过协议栈或单独监听处理。

思考题：在 WebSocket 协议中，Ping/Pong 控制帧可以插入到分片消息之间。如果一个应用正在发送一个大的分片消息（比如 100MB 的文件），中间插入 Ping 帧，接收方如何在重组分片的同时处理 Ping？这要求接收方在状态机中如何处理控制帧与数据帧的交叉？请从帧格式和状态机角度分析。
