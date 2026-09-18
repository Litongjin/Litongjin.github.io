---
title: "每日基础技术总结 · 2026-09-18 · TCP 四次挥手与半关闭状态"
date: 2026-09-18 08:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-18 · TCP 四次挥手与半关闭状态

## 📚 今日主题

> **TCP 四次挥手与半关闭状态**（网络基础）

### 1. 核心概念速览
核心概念速览

TCP 是全双工、面向连接、可靠字节流协议。连接由四元组 {src_ip, src_port, dst_ip, dst_port} 唯一标识。四次挥手是 TCP 连接终止过程，本质是双方独立关闭各自发送方向，并确保：1) 每一方向的数据都被完整传输并确认；2) FIN 消耗序列号，双方对序列号推进和最终 ACK 达成一致；3) 允许半关闭，即一端不再发送但仍可接收。

FIN 是 TCP 首部控制位，占用一个序列号。主动关闭方发送 FIN，进入 FIN_WAIT_1；收到对端 ACK 后进入 FIN_WAIT_2；收到对端 FIN 后回 ACK，进入 TIME_WAIT；等待 2MSL 后进入 CLOSED。被动关闭方收到 FIN 后回 ACK，进入 CLOSE_WAIT；应用调用 close/shutdown 后发送 FIN，进入 LAST_ACK；收到 ACK 后进入 CLOSED。

半关闭：调用 shutdown(fd, SHUT_WR) 仅发送 FIN，关闭本端写方向，本端仍可 read；对端 read 返回 0 表示 EOF。区别于 close(fd)：close 关闭读写两个方向并释放文件描述符，若接收缓冲区有未读数据可能触发 RST。

位置：传输层核心机制。HTTP/1.1 keep-alive、HTTP/2、WebSocket、gRPC、数据库连接池、反向代理、Kubernetes Service、Service Mesh、AI 推理服务长连接都依赖正确关闭。掌握它才能定位 CLOSE_WAIT 堆积、TIME_WAIT 过多、连接被 RST、优雅下线失败、fd 泄漏等问题。

前端对比：浏览器抽象掉 TCP 生命周期；fetch AbortController.abort() 通常触发 RST 或底层强制关闭；WebSocket close frame 是应用层关闭握手，底层仍走 TCP FIN/RST；Node.js 的 socket.end() 对应半关闭 FIN，socket.destroy() 对应强制关闭/RST。

### 2. 底层原理剖析
底层原理剖析

状态机：
CLOSED -> 主动 close/shutdown(SHUT_WR) -> 发送 FIN -> FIN_WAIT_1
FIN_WAIT_1 -> 收到 ACK -> FIN_WAIT_2
FIN_WAIT_1 -> 收到 FIN+ACK -> TIME_WAIT（同时关闭）
FIN_WAIT_2 -> 收到 FIN -> 回 ACK -> TIME_WAIT
TIME_WAIT -> 2MSL 超时 -> CLOSED
被动侧：
CLOSED -> 收到 FIN -> 回 ACK -> CLOSE_WAIT
CLOSE_WAIT -> 应用 close/shutdown(SHUT_WR) -> 发送 FIN -> LAST_ACK
LAST_ACK -> 收到 ACK -> CLOSED

为什么需要四次：TCP 连接两个方向独立。A 发送 FIN 只表示 A 无数据再发，不表示 A 不能再收。B 收到 FIN 后 TCP 层立即 ACK，防止 A 重传；但 B 应用可能还有数据要发，所以 B 的 FIN 需等应用决定。因此 ACK 与 FIN 通常分离。若 B 无数据且立刻关闭，可合并 FIN+ACK，报文数可少于 4。

序列号语义：FIN 消耗一个 seq。A 发送 FIN seq=x，B 回 ACK=x+1；B 发送 FIN seq=y，A 回 ACK=y+1。丢失则重传。TIME_WAIT 为 2MSL：确保最后 ACK 丢失时能重发；让旧连接报文在网络中消亡，避免新连接收到旧重复报文。2MSL 即报文最大生存时间两倍（一来一回）。

半关闭伪代码：
on receiving FIN(fin_seq):
  send ACK(fin_seq+1)
  state = CLOSE_WAIT
  deliver EOF to application read
  wait application close/shutdown WR
on application close/shutdown WR:
  if state == CLOSE_WAIT: send FIN; state = LAST_ACK
  else: send FIN; state = FIN_WAIT_1
on receiving ACK for FIN:
  if state == FIN_WAIT_1: state = FIN_WAIT_2
  if state == LAST_ACK: state = CLOSED
on receiving FIN(fin_seq):
  if state == FIN_WAIT_2: send ACK(fin_seq+1); state = TIME_WAIT; start 2MSL timer
  if state == FIN_WAIT_1: send ACK(fin_seq+1); state = TIME_WAIT  // 同时关闭

前端概念对比：WebSocket 的 close 帧类似应用层四次挥手：一端发 Close，另一端回 Close，再销毁 TCP；但 TCP 四次挥手在传输层，不依赖应用协议。HTTP/1.1 Connection: close 是服务端响应后关闭 TCP，属于完整关闭；Node.js 的 socket.end() 是半关闭，socket.destroy() 是强制关闭。TS interface 是编译期类型契约，运行时擦除；TCP 状态机是运行时内核协议状态，二者不在同一抽象层。Java interface 是运行时类型，可被反射/字节码引用。这个对比说明：不能把应用层契约等同于传输层状态。

### 3. 基础代码与实战验证
```text
基础代码与实战验证

Node.js 单进程演示半关闭。服务端必须设置 allowHalfOpen: true，否则 Node 收到 FIN 后会自动 end 写方向，无法演示半关闭。

const net = require('net');

const server = net.createServer({ allowHalfOpen: true }, (socket) => {
  console.log('server: new connection');
  socket.on('data', (chunk) => {
    console.log('server recv:', chunk.toString());
  });
  socket.on('end', () => {
    // 收到客户端 FIN：内核已回 ACK，服务端进入 CLOSE_WAIT；allowHalfOpen=true 阻止 Node 自动关闭写方向
    console.log('server: got FIN (CLOSE_WAIT), can still write');
    // 半关闭状态下写方向仍开放，发送响应后调用 end() 发送 FIN，进入 LAST_ACK
    socket.end('response after half-close');
  });
  socket.on('close', () => console.log('server: closed'));
});

server.listen(0, () => {
  const port = server.address().port;
  const client = net.createConnection({ port }, () => {
    console.log('client: connected');
    // end(data) 发送数据后立即发送 FIN：客户端进入 FIN_WAIT_1，收到 ACK 后进入 FIN_WAIT_2；写方向关闭，读方向仍开放
    client.end('request');
  });
  client.on('data', (chunk) => {
    console.log('client recv:', chunk.toString());
  });
  client.on('end', () => {
    // 收到服务端 FIN：客户端回 ACK，进入 TIME_WAIT
    console.log('client: got FIN (TIME_WAIT)');
  });
  client.on('close', () => {
    console.log('client: closed');
    server.close();
  });
});

验证命令：
node half_close.js
ss -tanp | grep <port>   # 观察 FIN_WAIT_2、CLOSE_WAIT、TIME_WAIT 等内核状态
sudo tcpdump -i lo -nn -S 'tcp port <port>'   # 观察 FIN、ACK 的 seq/ack 推进：FIN 消耗一个 seq

输出顺序体现半关闭：client 发送 request 后 FIN；server 读 EOF 后仍能写 response；client 收到 response 后再收到 FIN。若把服务端 allowHalfOpen 去掉，server 收到 FIN 后会立即关闭写方向，无法在半关闭后写回数据。
```

### 4. 常见误区与进阶思考
常见误区与进阶思考

误区一：close() 与 shutdown(SHUT_WR) 等价。close 关闭双向并释放 fd，若接收缓冲区有未读数据，内核可能发送 RST，跳过正常四次挥手，对端收到 ECONNRESET。Node.js 中 socket.end() 是半关闭写，socket.destroy() 是强制关闭；浏览器 fetch 的 AbortController.abort() 也可能触发 RST，而不是优雅 FIN。

误区二：看到四次挥手就认为必须四个报文；TIME_WAIT 是异常，应优化掉。实际上 ACK 与 FIN 可合并，被动方无数据可立刻关闭时可能只见 3 个报文。TIME_WAIT 是主动关闭方必要状态，保护新连接免受旧报文干扰并保证最后 ACK 可靠。SO_REUSEADDR 主要解决绑定问题，不等于消除 TIME_WAIT 对同一四元组的影响。CLOSE_WAIT 堆积通常是应用未 close，而非内核问题。

深度思考题：主动关闭方发送最后一个 ACK 后进入 TIME_WAIT。若该 ACK 丢失，对端处于 LAST_ACK 并重传 FIN。请描述双方状态迁移、计时器行为，以及为什么 TIME_WAIT 必须是 2MSL 而不是固定 1 个 RTT。进一步：设计一个基于 TCP 半关闭的应用协议时，如何区分对端正常关闭写方向与对端进程崩溃/网络中断？
