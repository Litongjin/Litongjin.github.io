---
title: "每日基础技术总结 · 2026-09-02 · TCP 四次挥手与半关闭状态"
date: 2026-09-02 08:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-02 · TCP 四次挥手与半关闭状态

## 📚 今日主题

> **TCP 四次挥手与半关闭状态**（网络基础）

### 1. 核心概念速览
TCP 四次挥手（Four-Way Handshake）是面向连接的可靠传输协议在断开连接时使用的状态机收敛过程，本质上是两个独立方向（客户端→服务端、服务端→客户端）各自执行一次半关闭（Half-Close）的报文交换。由于 TCP 连接是全双工管道，每一方向的数据流都有独立的序号空间和关闭语义，因此终止连接需要四个报文段：主动关闭方发送 FIN，被动关闭方回复 ACK，随后被动关闭方再发送自己的 FIN，主动关闭方回复 ACK。该机制解决的核心问题是：确保双方所有已发送数据都能被对端完整接收后再释放 socket 资源，避免数据截断与连接状态泄漏。它在整个计算机体系中的位置是传输层（Layer 4），是 HTTP/1.1、HTTP/2、gRPC、WebSocket 等应用层协议连接生命周期管理的基础。专业工程师必须掌握它，才能正确设计优雅关闭逻辑、排查 TIME_WAIT/CLOSE_WAIT 堆积、理解负载均衡器与反向代理的连接复用行为。

### 2. 底层原理剖析
在 ESTABLISHED 状态下，主动关闭方 A 调用 close 或 shutdown(SHUT_WR)，内核发送 FIN，状态迁移至 FIN_WAIT_1；被动关闭方 B 收到 FIN 后，内核立刻回复 ACK，同时向应用层提交 EOF（read 返回 0），B 状态迁移至 CLOSE_WAIT；A 收到该 ACK 后进入 FIN_WAIT_2。此时 A→B 方向已关闭，但 B→A 方向仍可继续传输数据，这就是半关闭状态。B 的应用层检测到 EOF 后，可以继续 write 剩余数据，最后调用 close 发送 FIN，B 进入 LAST_ACK；A 收到 FIN 后回复 ACK，并进入 TIME_WAIT，B 收到 ACK 后进入 CLOSED；A 等待 2MSL 后进入 CLOSED。四次挥手的根因是 ACK 与 FIN 的分离：被动关闭方无法在自己未决定关闭前预知未来是否还有数据要发，因此必须先 ACK 对方的 FIN，等应用层完成剩余写入后再独立发送自己的 FIN。与前端已有概念的对比：Node.js 的 net.Socket 在默认情况下收到 FIN 后会触发 'end' 事件，但若 allowHalfOpen 为 false，内核会在读端 EOF 后自动回 FIN，相当于被动方立即完成四次挥手；若 allowHalfOpen 为 true，则可在 'end' 后继续 write，等价于保留半关闭的写方向。这类似于前端中 ReadableStream 的 cancel 与 WritableStream 的 close 相互独立，或者与 EventEmitter 的 'end' 与 'close' 事件语义不同——'end' 表示读侧 EOF，'close' 表示底层资源彻底释放。

### 3. 基础代码与实战验证
```text
const net = require('net');

// 服务端：allowHalfOpen: true 表示收到客户端 FIN 后，不自动发送 FIN，
// 而是保留写方向，使得服务端可以继续向客户端发送数据。
const server = net.createServer({ allowHalfOpen: true }, (socket) => {
  socket.on('data', (chunk) => {
    // 收到请求数据，这里直接回显长度；实际可解析 HTTP。
    console.log('server received:', chunk.toString());
  });
  socket.on('end', () => {
    // 客户端 FIN 已到达，读方向结束；此时 TCP 处于半关闭状态。
    // 服务端仍可写，因此发送响应后主动调用 end() 发送本端 FIN。
    socket.end('HTTP/1.1 200 OK\r\nContent-Length: 2\r\n\r\nok');
  });
});
server.listen(8080);

const client = net.connect(8080, '127.0.0.1', () => {
  client.write('GET / HTTP/1.1\r\nHost: localhost\r\n\r\n');
  // 半关闭：发送 FIN，表示客户端写方向完成，但仍等待读取服务端响应。
  client.end();
});

client.on('data', (chunk) => {
  console.log('client received:', chunk.toString());
});
client.on('end', () => {
  // 服务端 FIN 到达，读方向结束，整个连接即将关闭。
  console.log('client: server FIN received');
  client.destroy();
});
```

### 4. 常见误区与进阶思考
误区一：认为四次挥手总是需要四个报文。如果被动关闭方在收到 FIN 时已经无数据可发（例如应用层早已调用了 close），内核可能将 ACK 与 FIN 合并为一个报文，从而变成三次报文交换（FIN, FIN+ACK, ACK）。不要以报文数量作为判断依据，而应以状态机和方向独立关闭为准。

误区二：混淆 close 与 shutdown/半关闭。在 Unix 中 close(fd) 是释放整个文件描述符引用，若多个进程/线程共享该 fd，只有引用计数归零才发送 FIN；而 shutdown(fd, SHUT_WR) 则直接使 TCP 发送 FIN，且不影响读方向。前端工程师常犯的类比错误是把 WebSocket 的 close() 当作 TCP close——实际上 WebSocket close 是应用层关闭帧，TCP 仍可能继续握手，而 Node.js 中 socket.destroy() 会立即丢弃缓冲区并可能发送 RST，不是优雅的 FIN。

进阶思考题：当主动关闭方进入 FIN_WAIT_2 后，若被动关闭方一直没有发送 FIN（例如应用进程挂死），主动关闭方会一直停留在 FIN_WAIT_2 吗？如果会，TCP 协议如何避免该状态无限残留？请结合内核实现与应用层超时策略回答。
