---
title: "每日基础技术总结 · 2025-12-21 · TCP 三次握手与四次挥手中的状态机流转及异常场景处理"
date: 2025-12-21 20:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2025-12-21 · TCP 三次握手与四次挥手中的状态机流转及异常场景处理

## 📚 今日主题

> **TCP 三次握手与四次挥手中的状态机流转及异常场景处理**（网络基础）

### 1. 核心概念速览
TCP 三次握手（Three-Way Handshake）与四次挥手（Four-Way Wave）是传输层建立可靠字节流连接与终止连接的协议状态机机制。其核心解决的是在不可靠的 IP 层之上，通过确认序列号（Seq）和ACK标志位同步双方初始序列号（ISN），确保全双工通道的数据有序、无丢失且能正确标识连接生命周期。该机制位于 OSI/TCP-IP 模型的传输层，是后端服务稳定性、前端 WebSocket/HTTP2 性能优化及分布式系统心跳检测的理论基石。工程师必须掌握其状态流转细节，以便深入理解延迟队列、重传超时（RTO）、半关闭状态对业务并发量的影响，以及避免常见的资源泄露问题。

### 2. 底层原理剖析
【三次握手 - 建立连接】
1. Client 发送 SYN=1, seq=x (SYN_SENT)。
2. Server 收到后，回复 SYN=1, ACK=1, seq=y, ack=x+1 (SYN_RCVD)。此时 Server 分配控制块（TCB）。
3. Client 收到后，发送 ACK=1, seq=x+1, ack=y+1 (ESTABLISHED)。Server 收到后进入 ESTABLISHED。
关键点：只有第三次握手的 ACK 到达 Server，连接才真正双向就绪。若第三次 ACK 丢失，Server 的重传 SYN-ACK 可导致 Client 重发 ACK，触发 'TCP 同步丢失' 但不会造成死锁。

【四次挥手 - 终止连接】
由于 TCP 是全双工的，关闭方向必须独立处理。
1. Client 发送 FIN=1, seq=u (FIN_WAIT_1)。
2. Server 收到 FIN，发送 ACK=1, ack=u+1 (CLOSE_WAIT)。此时 Server 仍可发送剩余数据，Client 处于 FIN_WAIT_2。
3. Server 应用层关闭 socket，发送 FIN=1, ACK=1, seq=v, ack=u+1 (LAST_ACK)。
4. Client 收到 FIN，发送 ACK=1, ack=v+1 (TIME_WAIT)。Server 收到 ACK 后彻底释放资源 (CLOSED)。

【状态机关键差异】
- ESTABLISHED: 数据传输阶段。
- TIME_WAIT: 主动关闭方最终状态。需坚持 2MSL (两倍最大报文生存时间)，确保最后一个 ACK 到达或过期；同时防止旧连接的重复报文干扰新连接。
- CLOSE_WAIT: 被动关闭方状态，若未及时处理会导致文件描述符泄漏。

【与前端概念对比】
前端 React/Vue 的状态管理通常是单向数据流或基于事件的通知机制，而 TCP 状态机是基于严格的时序和消息确认（Handshake/Ack）的双向同步协议。前端的 Promise Resolve/Reject 类似单次回调，而 TCP 连接的生命周期需要双方多次交互才能完成闭环，任何一步网络丢包都可能导致状态机的无限等待或错误分支。

### 3. 基础代码与实战验证
```text
// Node.js 原生 net 模块演示 TCP 状态流转的关键监听器
const net = require('net');

// 服务端逻辑
const server = net.createServer((socket) => {
  // 'connect' 触发表示三次握手成功，双方进入 ESTABLISHED
  console.log('Connection Established (ESTABLISHED)');

  // 'close' 通常伴随 TIME_WAIT 结束后调用，但需注意
  socket.on('close', () => {
    console.log('Socket Closed, resources released');
  });

  // 'drain' 提示底层缓冲已满，暂停写入，对应背压（Backpressure）
  socket.on('drain', () => {
    console.log('Buffer drained, can resume writing');
  });
});

server.listen(8080, () => {
  console.log('Server listening on port 8080');
});

// 客户端模拟挥手过程（需在外部 shell 测试或写完整示例）
// const client = new net.Socket();
// client.connect(8080, '127.0.0.1', () => {
//   console.log('Client Connected');
//   client.write('Hello');
//   client.end(); // 自动发送 FIN，进入 FIN_WAIT_1 -> FIN_WAIT_2 -> TIME_WAIT
// });
```

### 4. 常见误区与进阶思考
1. 【TIME_WAIT 的资源消耗误区】：开发者常误以为 TIME_WAIT 仅仅是‘等待’，忽视其对高并发场景下端口耗尽（Port Exhaustion）的影响。在高吞吐短连接架构中，大量客户端快速创建销毁连接会导致本地 ephemeral ports 被 TIME_WAIT 占用殆尽，无法发起新连接。解决方案包括开启 SO_REUSEADDR/SO_REUSEPORT 或在服务端采用连接池/长连接（如 HTTP Keep-Alive）。

2. 【CLOSE_WAIT 泄漏根源】：许多开发者只关注主动断开，却忽略被动关闭端（通常是服务器）在收到 FIN 后进入 CLOSE_WAIT 状态的维护责任。如果应用层代码未在业务逻辑完成后显式调用 close() 或 destroy()，socket 将一直停留在 CLOSE_WAIT，直至进程退出，导致句柄泄露和 OOM。这要求后端编码规范中必须包含finally块中的资源清理操作。

思考题：假设在一个弱网环境下，TCP 第四次挥手的最后一个 ACK 丢失了，此时服务端已经认为连接关闭并释放了资源，而客户端因未收到确认仍处于 TIME_WAIT 状态等待 2MSL。如果在 2MSL 期间，同一个源端口/IP 对再次发起一个新的 TCP 连接请求（SYN），服务端会如何处理这个新的 SYN？为什么 TCP 设计允许这种看似冲突的情况发生而不报错？
