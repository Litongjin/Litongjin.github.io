---
title: "每日基础技术总结 · 2026-08-22 · TCP 三次握手与 SYN Cookie"
date: 2026-08-22 08:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-22 · TCP 三次握手与 SYN Cookie

## 📚 今日主题

> **TCP 三次握手与 SYN Cookie**（网络基础）

### 1. 核心概念速览
TCP 三次握手是传输控制协议（TCP）在建立可靠双向字节流连接前，通过交换 SYN、SYN-ACK、ACK 三个报文段，同步双方初始序列号（ISN）并确认收发能力的过程。其本质是建立连接双方的状态机同步，而非简单的“打招呼”。它解决的核心问题是：在不可靠的 IP 网络上，如何在不预先共享任何状态的情况下，安全地分配序列号空间并防止旧连接的重复报文干扰新连接。SYN Cookie 是服务器端针对 SYN Flood 拒绝服务攻击的一种无状态防护机制：服务器在收到 SYN 时不立即分配连接控制块（TCB），而是基于五元组、时间戳和密钥计算出一个 Cookie 值作为 SYN-ACK 的初始序列号；仅当客户端返回 ACK 且 Cookie 校验通过时，才真正分配资源。该机制将资源分配延迟到连接建立确认之后，从而避免攻击者用大量伪造 SYN 耗尽服务器内存。在计算机体系中，TCP 位于网络栈的传输层，介于 IP 网络层与应用层之间；它是 HTTP、WebSocket、数据库协议等所有可靠通信的基石。专业工程师必须掌握握手状态机和 SYN Cookie 细节，因为这是理解连接超时、性能调优、网络诊断以及抵御常见攻击的基础，也是从应用层工程师走向系统级工程师的必经门槛。

### 2. 底层原理剖析
三次握手的底层机制是序列号同步与状态机转移。客户端初始状态 CLOSED，服务器监听状态 LISTEN。过程如下：
1. 客户端发送 SYN，seq=x，进入 SYN_SENT。该报文不携带应用数据，但占用一个序列号。
2. 服务器收到 SYN，若接受连接，则发送 SYN-ACK，seq=y，ack=x+1，进入 SYN_RCVD。服务器在此刻通常分配 TCB 并进入半连接队列（Backlog Queue）。
3. 客户端收到 SYN-ACK，验证 ack 等于自己发送的 x+1，则发送 ACK，seq=x+1，ack=y+1，进入 ESTABLISHED。服务器收到 ACK 后，验证 ack 正确，将连接从半连接队列移入全连接队列，进入 ESTABLISHED。
关键本质：序列号 x 和 y 是两个独立的方向上的初始序号，握手完成的是两个方向的序列号同步。握手的第三个报文 ACK 如果丢失，服务器仍会进入 ESTABLISHED（因为只要收到过 SYN-ACK 且客户端已经能发送数据），但客户端会重传 ACK。
SYN Cookie 机制在服务器收到 SYN 时不创建半连接，而是计算：
  cookie = hash(源IP, 源端口, 目的IP, 目的端口, 服务器密钥, 时间戳分片)
将 cookie 作为 SYN-ACK 的序列号 y 返回。服务器不保存任何状态。当客户端返回 ACK 时，服务器根据 ACK 中的 ack 值（即 y+1，等于 cookie+1）反向验证 cookie 的有效性，并重建连接参数。
与前端概念对比：前端工程师熟悉的 TypeScript 接口是编译期的结构约束，与运行时无关；而 TCP 握手是运行时的协议状态机，每个状态和报文都有真实的内存和网络开销。更相近的对比是 Promise 的状态机：pending → fulfilled/rejected，但 TCP 状态机具有双向性，且涉及超时重传、序列号回绕等更复杂的问题。前端中“幂等”概念与 TCP 序列号作用类似——通过序列号识别重复报文，但 TCP 还需要处理流量控制和拥塞控制。理解这些差异有助于从应用层思维转向协议层思维。

### 3. 基础代码与实战验证
```text
由于 TCP 三次握手和 SYN Cookie 由操作系统内核实现，不依赖应用代码，因此这里给出基于 socket 的最小验证代码，以及内核处理 SYN Cookie 的伪代码。

# Python 示例：模拟 TCP 服务端，观察三次握手后的连接状态（依赖内核完成握手）
import socket

# 创建监听 socket
server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server_socket.bind(('0.0.0.0', 8080))  # 绑定端口
server_socket.listen(5)  # 指定内核维护的全连接队列长度

conn, addr = server_socket.accept()  # 阻塞等待，直到三次握手完成并进入全连接队列
# 此时连接已 ESTABLISHED，内核已完成序列号同步，应用层无需感知握手过程
print('Connection from', addr)
data = conn.recv(1024)  # 接收数据，底层依赖已建立的可靠流通道
conn.sendall(b'ACK')    # 发送数据，同样依赖 TCP 的序列号和确认机制
conn.close()

# 内核处理 SYN Cookie 的伪代码（以 Linux 为例）
# 在 tcp_v4_conn_request 中，若半连接队列满且启用 SYN Cookie：
#   cookie = cookie_v4_init_sequence(sk, skb)
#     其中使用源/目的地址、端口、mss、一个全局密钥和 jiffies 分片计算哈希
#   发送 SYN-ACK，序列号设为 cookie，不分配 request_sock 结构
# 在 tcp_v4_rcv 收到 ACK 后，调用 cookie_v4_check()
#   根据 ACK 的 ack 号（cookie+1）逆推哈希，若匹配且时间戳在有效窗口内，则创建 request_sock 并完成握手

关键注释：
- listen(5) 中的 5 是内核全连接队列长度，不是半连接队列。半连接队列由内核参数 tcp_max_syn_backlog 控制，但启用 SYN Cookie 后，半连接队列的作用被绕过。
- recv() 阻塞直到收到数据，这是 TCP 可靠字节流的表现；如果应用层不调用 recv，数据仍会由内核缓冲。
```

### 4. 常见误区与进阶思考
误区 1：认为三次握手的目的是“确认双方具备收发能力”。实际上，三次握手更本质的目的是同步初始序列号，因为序列号是可靠传输的基础。两次握手无法保证旧连接的重复 SYN 不会导致新连接建立，因为服务器无法确认客户端是否收到了自己的 SYN-ACK。虽然四次握手可以更彻底，但三次已足够（因为 ACK 可以捎带序列号）。
误区 2：认为 SYN Cookie 是“紧急防护”或“非正常模式”，只用于攻击时。实际上，Linux 在 tcp_syncookies 参数设为 1 时，当半连接队列满时自动启用；设为 2 时无条件启用。它牺牲了部分 TCP 扩展功能（如窗口缩放），但避免了资源耗尽，是生产环境的重要防御机制。理解其代价有助于正确配置和调优。
思考题：在 SYN Cookie 机制下，如果攻击者伪造大量源 IP 发起 SYN，并立刻回复伪造的 ACK（携带正确的 cookie），服务器是否会为每个 ACK 分配连接资源？如果是，如何防止这种资源消耗？如果不是，请解释 cookie 验证中哪个环节阻止了这种攻击。
