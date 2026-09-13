---
title: "每日基础技术总结 · 2026-09-13 · TCP 四次挥手与半关闭状态"
date: 2026-09-13 08:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-13 · TCP 四次挥手与半关闭状态

## 📚 今日主题

> **TCP 四次挥手与半关闭状态**（网络基础）

### 1. 核心概念速览
TCP 四次挥手是传输控制协议在结束全双工连接时，通过交替发送 FIN 与 ACK 控制段，独立关闭两个方向数据传播的机制。本质是把一条 TCP 连接视为两个单向字节流管道，每个方向必须单独关闭；半关闭（half-close）即一方发送 FIN 后不再发送但继续接收的状态。它解决的问题是：在允许数据双向同时流动的可靠传输层，如何优雅终止连接而不丢失对端尚未送达的数据。四次挥手位于 TCP/IP 传输层，是可靠传输语义的收尾部分；它与三次握手共同构成 TCP 连接生死完整闭环。专业工程师必须掌握它，因为反向代理、服务发现、优雅停机、连接池回收、消息队列确认等场景都依赖对 FIN、ACK、TIME_WAIT 状态的精确判断，且很多线上 bug 是由关闭时序错误导致的。

### 2. 底层原理剖析
状态机：
ESTABLISHED → (A 调用 shutdown/close) → FIN_WAIT_1：A 发送 FIN，停止发送数据，但仍可接收。
B 收到 FIN → CLOSE_WAIT：B 应用层通过 recv 返回 EOF 感知到 A 不再发送；若 B 还需要发送数据，此刻正是半关闭窗口。B 回复 ACK → A 收到 ACK，进入 FIN_WAIT_2。
B 发送完所有数据后调用 close，发送 FIN → LAST_ACK：B 等待 A 的最终 ACK。
A 收到 FIN，回复 ACK，进入 TIME_WAIT：等待 2MSL 后关闭；B 收到 ACK 后立即 CLOSED。
关键机制：被动关闭方不会在收到 FIN 后立即关闭发送方向，必须由应用层主动 close/shutdown，否则连接停留在 CLOSE_WAIT。TIME_WAIT 长度为 2MSL，一是保证最后 ACK 丢失时能重传，二是防止旧连接中残留分组被新连接误收。
与前端已有概念类比：HTTP/1.1 的 Connection: close 是应用层对整个连接的整体关闭语义，而 TCP 四次挥手是传输层对两个独立方向的精细化关闭，两者处于不同协议层。这类似 Java 接口（运行时多态契约）与 TypeScript 接口（编译期结构契约）共享了“接口”一词，但生效时机和底层机制完全不同；因此不能把前端 fetch/abort 的一体化取消直接等同于 TCP 半关闭。

### 3. 基础代码与实战验证
```text
import socket

srv = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
srv.bind(('127.0.0.1', 0))
srv.listen(1)
port = srv.getsockname()[1]

cli = socket.create_connection(('127.0.0.1', port))
conn, _ = srv.accept()

# 客户端发送请求后通过 shutdown 发送 FIN，但仍保留读方向
cli.sendall(b'GET / HTTP/1.0\r\n\r\n')
cli.shutdown(socket.SHUT_WR)

# 服务端持续接收，直到 recv 返回 b''（收到 FIN）
data = b''
while True:
    chunk = conn.recv(4096)
    if not chunk:
        break
    data += chunk

# 服务端此时仍可发送响应，发送完成后 close 触发自己的 FIN
conn.sendall(b'HTTP/1.0 200 OK\r\nContent-Length: 2\r\n\r\nOK')
conn.close()

# 客户端收完整响应后，recv 返回 b''，半关闭结束
response = b''
while True:
    chunk = cli.recv(4096)
    if not chunk:
        break
    response += chunk

cli.close()
srv.close()
print(response.decode())

注释：cli.shutdown(socket.SHUT_WR) 使内核发出 FIN，此后 cli 仍可 recv，这是半关闭的直观验证。
```

### 4. 常见误区与进阶思考
1. 把 close() 等同于双向立刻断开。实际上 close() 使本地 socket 变为 FIN_WAIT_1 并发送 FIN，但内核默认会先发送发送缓冲区中的剩余数据；而 shutdown(SHUT_WR) 才是明确半关闭。很多工程师在实现自定义传输协议时只调 close，没有先读对端最后的响应，导致对端数据还没发完本地就关闭，出现 Connection reset by peer 或丢失数据。
2. 忽视主动关闭方进入 TIME_WAIT 的时间窗。在 TIME_WAIT 内，相同四元组不能立即被 bind/connect 复用；服务端大量短连接重建时会积压 TIME_WAIT，如果不设置 SO_REUSEADDR 或改成长连接，就会端口不可用。高级工程师要区分 TIME_WAIT 是协议正确性的一部分，不是单纯资源泄漏。

思考题：主动关闭方调用 shutdown(SHUT_WR) 后处于 FIN_WAIT_2，如果对端一直不关闭也不发送数据，这个连接会一直存在吗？你认为应该靠什么机制兜底？
