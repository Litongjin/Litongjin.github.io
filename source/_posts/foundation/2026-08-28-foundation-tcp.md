---
title: "每日基础技术总结 · 2026-08-28 · TCP 四次挥手与半关闭状态"
date: 2026-08-28 08:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-28 · TCP 四次挥手与半关闭状态

## 📚 今日主题

> **TCP 四次挥手与半关闭状态**（网络基础）

### 1. 核心概念速览
TCP 四次挥手是 TCP 连接关闭阶段的有限状态机过程，其本质是拆除一条全双工连接上的两条独立单向数据通道。TCP 用序列号与确认号保证字节流的可靠传输，因此『关闭』也必须以同样可靠的方式终止双向数据流。四次挥手解决的问题是：在不对连接中的遗留数据做静默丢弃的前提下，让双方都明确知道对方不会再发送数据，且自己可以安全释放内核中的连接资源。机制是：主动关闭方发送 FIN 表示其发送方向终止；被动关闭方用 ACK 确认这个方向关闭，之后若自身仍有数据，仍可继续发送；待被动方数据发送完毕后，也发送 FIN 关闭它的发送方向；主动方回复最后 ACK 并进入 TIME_WAIT，等待 2MSL 以确保最后 ACK 可被重传且过往报文段不会污染新的连接。四次挥手位于传输层 TCP 状态机中，是 Nginx、Node.js、网关、负载均衡与后端服务高并发连接调优的基础。专业工程师必须掌握它，因为生产环境最常见的 CLOSE_WAIT 堆积、TIME_WAIT 过多、连接被 RST 等问题，全部源自对关闭语义和状态迁移的误解。

### 2. 底层原理剖析
TCP 是全双工协议，连接的生命周期中，数据在两个方向上独立传输。因此『关闭』必须分别完成两件事：停止从 A 到 B 的数据，停止从 B 到 A 的数据。四次挥手本质就是这两个单向关闭过程的叠加。

精确时序如下，设 A 为主动关闭方，B 为被动关闭方：
1. A 调用 shutdown(SHUT_WR) 或 close()，应用层不再写数据。内核清空发送缓冲区后发送 FIN(seq=p)，A 进入 FIN_WAIT_1。FIN 会消费一个序列号，所以后续 ACK 是对端回复 ack=p+1。
2. B 收到 FIN(seq=p)，TCP 协议栈自动回复 ACK(ack=p+1)，B 进入 CLOSE_WAIT。A 收到这个 ACK 后进入 FIN_WAIT_2。此时 A->B 方向已经关闭，B->A 方向仍然打开。A 不能再发送数据，但可以继续接收；B 仍可以继续发送数据。这个状态就是半关闭（Half-Close）。
3. B 将自己的数据全部发送完毕后，调用 close() 或 shutdown(SHUT_WR)，内核发送 FIN(seq=q)，B 进入 LAST_ACK。
4. A 收到 FIN(seq=q)，回复 ACK(ack=q+1)，进入 TIME_WAIT。B 收到 ACK 后立即进入 CLOSED。A 在 TIME_WAIT 中等待 2MSL（Maximum Segment Lifetime）后才进入 CLOSED。

半关闭的核心机制：TCP 允许一端在发送 FIN 后继续接收对端数据。只要调用的是 shutdown(SHUT_WR) 而不是 close()，本端 socket 的读方向仍然有效。close() 是关闭整个 socket，shutdown(SHUT_WR) 是单方向关闭。这是正确理解四次挥手的关键。

与前端知识体系对比：HTTP/2 的 DATA 帧或 HEADERS 帧上的 END_STREAM 标志正是把 TCP 半关闭语义搬到了应用层——每个方向独立结束，一端发了 END_STREAM 后仍然可以继续接收另一端数据。Node.js 的 net.Socket 的 allowHalfOpen 选项也是直接暴露 TCP 半关闭能力。前端熟悉的 HTTP 请求-响应是单次循环，往往看不见操作系统层面的 FIN_WAIT/CLOSE_WAIT 状态，但代理服务器、反向代理和长连接服务都是基于这个状态机工作的。

### 3. 基础代码与实战验证
```text
import socket

# ============ server.py ============
# 服务器端：验证客户端半关闭后，服务器仍可以向客户端发送数据
srv = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
srv.bind(('127.0.0.1', 9000))
srv.listen(1)
conn, _ = srv.accept()

# 不断接收客户端数据，直到收到 FIN。
# recv() 返回 b'' 表示 TCP 层收到 FIN，即客户端发送方向已经关闭。
# 但客户端接收方向仍然打开，服务器此刻处于 CLOSE_WAIT 状态。
data = b''
while True:
    chunk = conn.recv(4096)
    if not chunk:
        break
    data += chunk

# 服务器仍可以向客户端写数据。
# shutdown(SHUT_WR) 触发发送 FIN，服务器进入 LAST_ACK；
# 收到客户端的最终 ACK 后进入 CLOSED。
conn.sendall(b'ack:' + data)
conn.shutdown(socket.SHUT_WR)
conn.close()

# ============ client.py ============
import socket

s = socket.create_connection(('127.0.0.1', 9000))
s.sendall(b'hello')

# 半关闭：发送 FIN，声明“我不再发送数据”。
# socket 进入 FIN_WAIT_1，收到服务器 ACK 后进入 FIN_WAIT_2。
# 这个状态下客户端仍可以 recv()，正好验证半关闭语义。
s.shutdown(socket.SHUT_WR)

# 接收服务器后续发来的数据。
# 当服务器也发送 FIN 后，客户端回复 ACK 并进入 TIME_WAIT。
reply = s.recv(4096)
print(reply)

# close() 释放客户端 socket，TIME_WAIT 由内核在等待 2MSL 后自动回收。
s.close()
```

### 4. 常见误区与进阶思考
误区一：认为收到 FIN 就代表连接已经关闭，本端不能再发送数据。实际上 FIN 只表示对方发送方向终止，本端发送方向依然有效。在 CLOSE_WAIT 状态下，如果还有响应数据要发送，仍然可以继续 send()。只有调用 close() 或 shutdown(SHUT_WR)，本端发送方向才会关闭。贸然 close() 会立刻关闭读写两个方向，如果有未读数据还可能导致内核发送 RST，破坏有序关闭。

误区二：认为 TIME_WAIT 是垃圾状态，应该尽可能规避或暴力清除。TIME_WAIT 是 TCP 可靠性的基石：它保证最后一个 ACK 丢失时可以重传，并保证旧连接中的报文段在 2MSL 后全部消亡，不会混入相同四元组的新连接。TIME_WAIT 大量堆积通常意味着应用层连接池复用不足或服务端主动关闭频繁，正确方向是优化连接复用，而不是修改内核参数消灭 TIME_WAIT。真正需要警惕的是 CLOSE_WAIT 大量堆积，那说明被动关闭方应用层忘记关闭 socket。

思考题：主动关闭方 A 收到 B 的 FIN 后发送最后一个 ACK 并进入 TIME_WAIT。如果这个 ACK 丢失，B 会停留在 LAST_ACK 并重发 FIN。请问 A 在 TIME_WAIT 期间收到重发的 FIN 时会如何响应？当 2MSL 结束后 A 进入 CLOSED，而 B 如果仍收不到 ACK，系统最终会怎样恢复？请结合最后 ACK 的不可靠性和 TCP 的保活/超时机制给出完整状态路径。
