---
title: "每日基础技术总结 · 2026-09-17 · TCP 三次握手与 SYN Cookie"
date: 2026-09-17 08:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-17 · TCP 三次握手与 SYN Cookie

## 📚 今日主题

> **TCP 三次握手与 SYN Cookie**（网络基础）

### 1. 核心概念速览
TCP 三次握手是传输层连接建立阶段的状态同步过程：客户端与服务器交换 SYN、SYN+ACK、ACK 三个控制段，确认双方初始序列号 ISN，协商 MSS、SACK、窗口缩放、时间戳等选项，最终双方进入 ESTABLISHED。其本质不是『确认双方能收发』这么简单，而是在不可靠信道上可靠地建立双向字节流：同步序列号、确认对端已收到自己的 ISN、防止历史连接初始化导致错误连接。SYN Cookie 是服务器端对抗 SYN Flood 的无状态连接建立机制：当半连接队列满或 tcp_syncookies 强制启用时，服务器不保存 request_sock/TCB，而是用哈希 H(密钥, 源IP, 目的IP, 源端口, 目的端口, 时间, MSS索引) 计算 32 位 cookie，作为 SYN+ACK 的 seq 发出；收到第三次 ACK 时验证 ack-1 并重建 TCB。它解决半连接队列资源被伪造源 IP 的 SYN 占满，导致合法连接无法建立的问题。位置：TCP 是 HTTP/1.1、HTTP/2、TLS、WebSocket、gRPC、数据库连接、AI 分布式训练通信的底层承载。前端工程师必须掌握：fetch/XHR/WebSocket 的延迟、连接复用、并发限制、TIME_WAIT、负载均衡回源、CDN 边缘连接都受握手与内核队列影响。

### 2. 底层原理剖析
1. 三次握手状态机与报文：
C -> S: SYN, seq=x, options(MSS, SACK_PERM, WS, TS)
S -> C: SYN+ACK, seq=y, ack=x+1, options
C -> S: ACK, seq=x+1, ack=y+1
此后双方进入 ESTABLISHED。ISN 必须随机化，防止旧报文注入。两次不够：服务器无法确认客户端收到 SYN+ACK；四次可合并为三次。

2. 内核队列：LISTEN 后收到 SYN，内核创建 request_sock 放入 SYN queue（半连接队列），回复 SYN+ACK。收到 ACK 后移入 accept queue（全连接队列），等待 accept()。两个队列有上限，溢出时丢弃或触发 SYN Cookie。

3. SYN Cookie 伪代码：
if syn_queue_full or tcp_syncookies == 1:
  cookie = H(secret, saddr, daddr, sport, dport, time, mss_index)
  send SYN+ACK(seq=cookie, ack=client_seq+1)
收到 ACK:
  if valid_cookie(ack-1, saddr, daddr, sport, dport, now):
    重建 TCB，进入 ESTABLISHED
  else: drop
Linux 实现把时间低 5 位、MSS 索引等编码进 cookie，验证时检查时间窗口与哈希。

4. 与前端概念对比：
fetch/XHR/WebSocket 是应用层 API，TCP 握手是内核传输层行为。HTTP/1.1 keep-alive 复用已建立连接，避免重复握手；HTTP/2 多路复用同一 TCP 连接，但仍受 TCP 层队头阻塞。WebSocket 先完成 TCP 三次握手，再发 HTTP Upgrade。
TS interface 与 Java interface：TS 接口是编译期结构类型，运行时擦除；TCP 握手是运行时状态机，有真实报文、序列号、定时器、内核队列。Java 接口有运行时方法表，类似内核 TCB 有真实状态。SYN Cookie 类似无状态 JWT：服务端不存 session，用哈希/签名验证；但 JWT 可重放，SYN Cookie 绑定四元组与时间窗口，且只用于握手。

### 3. 基础代码与实战验证
```text
服务端 server.py：
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)  # 创建 TCP 套接字
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)  # 地址复用
s.bind(('127.0.0.1', 8080))  # 绑定监听地址
s.listen(1)  # 进入 LISTEN，内核维护 SYN 队列与 accept 队列
conn, addr = s.accept()  # 从 accept 队列取出已完成三次握手的连接
print('accepted', addr)
conn.close()
s.close()

客户端 client.py：
import socket
c = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
c.connect(('127.0.0.1', 8080))  # 阻塞直到内核完成 SYN -> SYN+ACK -> ACK
info = c.getsockopt(socket.IPPROTO_TCP, socket.TCP_INFO, 1)  # Linux 读取 TCP_INFO
print('tcp_state', info[0])  # tcpi_state=1 表示 ESTABLISHED
c.close()

验证三次握手：
先执行 tcpdump -i lo -n 'tcp port 8080' -S
再运行 server.py，然后 client.py。tcpdump 应出现：
Flags [S], seq 客户端ISN
Flags [S.], seq 服务器ISN, ack 客户端ISN+1
Flags [.], ack 服务器ISN+1
这对应三次握手。

验证 SYN Cookie：
sysctl -w net.ipv4.tcp_syncookies=1
sysctl -w net.ipv4.tcp_max_syn_backlog=1
用 hping3 -S -p 8080 --flood 127.0.0.1 发送大量 SYN，同时 ss -lnt 观察队列；服务器仍应能接受合法 client.py 连接。
```

### 4. 常见误区与进阶思考
误区1：认为三次握手是为了确认双方收发能力。确认收发能力两次即可，三次的核心是同步 ISN 并确认对方已收到自己的 ISN，防止历史连接请求造成错误连接。在 SYN Cookie 下服务器不保存半连接状态，第三次 ACK 通过 cookie 验证并重建 TCB。

误区2：认为启用 SYN Cookie 没有代价。SYN Cookie 会丢失部分 TCP 选项（MSS、窗口缩放、SACK、时间戳等），可能降低吞吐与延迟；且无状态意味着无法保存扩展协商，必须依赖编码或回退。生产环境通常结合 syncache、队列调优、防火墙，而非长期强制开启。

思考题：在 SYN Cookie 模式下，服务器收到第三次 ACK 后验证 cookie 并创建 TCB。若攻击者伪造大量 ACK，携带不同四元组和猜测的 cookie，服务器如何避免 CPU 被哈希验证耗尽？进一步：若客户端在第三次 ACK 中携带 payload（TCP Fast Open），服务器在验证 cookie 前如何处理该数据？这检验对无状态握手、序列号验证、TFO 与 SYN Cookie 交互的理解。
