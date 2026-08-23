---
title: "每日基础技术总结 · 2026-08-23 · TCP 四次挥手与半关闭状态"
date: 2026-08-23 08:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-23 · TCP 四次挥手与半关闭状态

## 📚 今日主题

> **TCP 四次挥手与半关闭状态**（网络基础）

### 1. 核心概念速览
TCP 四次挥手（Four-Way Handshake）是 TCP 在 ESTABLISHED 状态下终止一条全双工连接的过程。其本质是两条独立单向数据流的分别关闭：每一端通过发送 FIN 表示“我的发送方向已结束”，对端回 ACK 确认；由于两个方向需要独立确认，因此正常需要四个报文段。半关闭（Half-Close）指连接的一端已经关闭了发送方向，但仍保留接收能力，使对端能继续发送剩余数据。该机制解决的核心问题是：在可靠字节流上优雅地终止连接，确保双方已发送的数据全部被对端接收，避免数据丢失。TCP 位于网络体系中的传输层，是面向连接、可靠传输的基础，为 HTTP、WebSocket 等上层协议提供语义。专业工程师必须掌握，因为连接终止是分布式系统故障的常见来源，例如服务端大量 CLOSE_WAIT 堆积、TIME_WAIT 导致的端口复用问题，均源于对四次挥手状态机的理解不足。

### 2. 底层原理剖析
四次挥手状态机如下（A 为主动关闭方，B 为被动关闭方）：初始 ESTABLISHED。A 发送 FIN，进入 FIN_WAIT_1；B 收到后回复 ACK，进入 CLOSE_WAIT；A 收到 ACK 进入 FIN_WAIT_2。此时 B 仍可向 A 发送数据，A 仍可接收，这就是半关闭状态。B 发送完数据后，发送 FIN 进入 LAST_ACK；A 收到 B 的 FIN 后回复 ACK 并进入 TIME_WAIT；B 收到 ACK 进入 CLOSED。A 在 TIME_WAIT 等待 2*MSL 后进入 CLOSED。为什么不是三次：因为 ACK 和 FIN 在正常情况下不能合并。只有当被动关闭方在收到 FIN 时恰好也无数据要发，才可能将 ACK 和 FIN 合并，形成三次挥手，但这只是优化，不改变语义。TCP 是全双工，两条方向相互独立，因此终止必须分别确认。与前端概念对比：这类似 Node.js 中 Duplex 流，readable 和 writable 可以独立关闭，调用 readable.destroy() 不影响 writable；而 HTTP/1.1 的 Connection: close 是语义上的“整体关闭”，不存在半关闭的选项。前端工程师熟悉的 WebSocket Close 握手虽然也是双向的，但标准要求双方都发送 Close 帧后连接才算关闭，且不允许单侧保持打开用于继续发送业务数据，和 TCP 的 half-close 粒度不同。

### 3. 基础代码与实战验证
```text
以 Python socket 演示半关闭（真实代码）：

    import socket

    # 服务端：绑定端口并 accept
    srv = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    srv.bind(('127.0.0.1', 8000))
    srv.listen(1)
    conn, addr = srv.accept()

    # 客户端：连接并发送数据
    cli = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    cli.connect(('127.0.0.1', 8000))
    cli.sendall(b'hello')

    # 客户端半关闭：只关闭发送方向，内核发送 FIN，但仍可接收服务端数据
    cli.shutdown(socket.SHUT_WR)

    # 服务端读取到 EOF（recv 返回 b''），意味着客户端发送方向已关闭
    assert conn.recv(1024) == b'hello'
    assert conn.recv(1024) == b''

    # 服务端发送响应后关闭连接，此时发送 FIN
    conn.sendall(b'world')
    conn.close()

    # 客户端仍能收到数据（因为只关闭了发送方向）
    assert cli.recv(1024) == b'world'

关键行注释：cli.shutdown(socket.SHUT_WR) 只发送 FIN，不销毁 socket 文件描述符，后续 recv 仍可读；conn.recv(1024) == b'' 表示对端 FIN 已到达，接收方向读到 EOF，但服务端仍可以继续向对端发送数据。如果改用 cli.close()，则发送和接收都关闭，后续 recv 会抛异常。实际四次挥手由内核协议栈完成，用户态只需调用 shutdown 或 close。
```

### 4. 常见误区与进阶思考
常见误区 1：认为四次挥手必须严格是四个报文段。实际上如果被动关闭方在收到 FIN 后无数据要发，内核可能把 ACK 和 FIN 合并发送，变成三次握手。但这不影响半关闭语义，正常状态机仍按四步推导。误区 2：混淆 close 和 shutdown。close 立即释放文件描述符，若仍有未读数据或对端发送 FIN 尚未处理，可能导致 RST；shutdown 提供三个粒度：SHUT_RD、SHUT_WR、SHUT_RDWR。生产环境大量 CLOSE_WAIT 堆积往往是因为服务端收到 FIN 后没有正确调用 close，说明应用层没有感知半关闭。思考题：若 A 主动发送 FIN 后进入 FIN_WAIT_2，B 收到 FIN 后进入 CLOSE_WAIT，但 B 一直不发送 FIN，请问此时 A 和 B 各自还能发送数据吗？各自的接收方向状态如何？如果 B 永远不发送 FIN，A 的 FIN_WAIT_2 会持续多久？这需要结合半关闭和 TCP 的定时器机制回答。
