---
title: "每日基础技术总结 · 2026-06-05 · TCP 三次握手与 SYN Cookie"
date: 2026-06-05 08:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-06-05 · TCP 三次握手与 SYN Cookie

## 📚 今日主题

> **TCP 三次握手与 SYN Cookie**（网络基础）

### 1. 核心概念速览
TCP 三次握手是传输控制协议（TCP）在连接建立阶段使用的同步机制，其本质是通过交换 SYN、SYN-ACK、ACK 三个报文段，在不可靠的 IP 网络上协调通信双方的初始序列号（ISN），从而确立双向数据流的起始状态。它解决的核心问题是：在没有全局时钟、报文可能乱序或重复的网络环境中，如何安全地同步双方序列号，防止旧连接的重复报文干扰新连接。机制上，客户端发送 SYN（seq=x），服务端回复 SYN-ACK（seq=y, ack=x+1），客户端再发送 ACK（seq=x+1, ack=y+1），至此双方均确认了对方的接收能力和自身发送的起始序号。在整个计算机体系中的地位：TCP 是传输层的事实标准，三次握手是 TCP 连接生命周期中第一个状态迁移过程，是可靠传输、流量控制、拥塞控制的前提。后端工程师理解它能避免在连接池、超时调优、DDoS 防护等场景中做出错误判断；AI 工程师则需依赖分布式系统的网络稳定性，掌握握手机制有助于理解负载均衡、服务网格中的连接管理。

### 2. 底层原理剖析
底层运行机制：TCP 连接建立需要双方各自维护一个发送序号和接收序号。三次握手本质是四次状态更新的折叠：第一次握手（SYN）告知对方我的 ISN 为 x，并期望收到对方的 ISN；第二次握手（SYN-ACK）告知对方我的 ISN 为 y，并确认已收到 x；第三次握手（ACK）确认已收到 y，并携带 x+1。这样双方都确认了“我的发送对方能收到”和“对方的发送我能收到”。若使用两次握手，服务端无法区分“请求是否过时”——一个迟到的 SYN 会让服务端误建连接并分配资源，导致资源浪费和序列号错乱。三次握手的核心是让双方都确认对方的 ISN，并确保双方的接收窗口和 MSS（最大报文段长度）等参数达成一致。SYN Cookie 是应对 SYN Flood 攻击的一种无状态握手机制：服务端不立即分配连接控制块（TCB），而是将连接参数（如 MSS、时间戳）通过密钥散列（通常为 SipHash 或 MD5）编码到初始序列号（即 cookie）中，在 SYN-ACK 中返回。若客户端是真实的，会返回 ACK 携带 cookie+1；服务端校验 cookie 的合法性和时效性后才分配 TCB，从而避免半连接队列被占满。与前端概念对比：Java 的接口与 TS 的接口本质区别在于前者是运行期类型约束（字节码层面存在），后者是编译期结构检查（类型擦除后无运行期痕迹）。而三次握手与 HTTP 的“请求-响应”不同——HTTP 是基于 TCP 的应用层协议，每次请求在 TCP 连接上可以是连续多个数据段；三次握手是 TCP 的状态机迁移，而非请求语义。前端开发者容易将“连接”误解为“一次请求”，实际上一个 TCP 连接可承载多个 HTTP 请求（Keep-Alive），而三次握手只发生一次。另一个对比：前端中的事件循环（Event Loop）是单线程非阻塞机制，而 TCP 握手是内核协议栈中状态机的并发流转，两者在“状态管理”层面有相似性——都需要处理超时和重传，但 TCP 的握手依赖系统调用（connect/accept）和内核回调，而非用户态事件。

### 3. 基础代码与实战验证
```text
以下用 Python 的 socket 库演示客户端和服务端三次握手的可见状态（注意：三次握手由内核自动完成，但可通过 TCP_INFO 观察状态迁移）。

# 服务端代码
import socket, struct, time

# 创建一个 IPv4/TCP 套接字
server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
# 允许地址复用，避免 TIME_WAIT 状态导致绑定失败
server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server.bind(('127.0.0.1', 8080))
# 监听队列长度，当全连接队列满时，多余连接将被丢弃
server.listen(5)

# 客户端代码（同一进程内模拟，实际应在另一进程）
client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
# connect 触发三次握手：内核发送 SYN，等待 SYN-ACK，再发送 ACK
client.connect(('127.0.0.1', 8080))

# 服务端 accept 从全连接队列取回一个已完成握手的连接
conn, addr = server.accept()
print('accept 返回，连接已建立')

# 观察 TCP 状态：通过 getsockopt 获取 TCP_INFO（Linux 专属）
def tcp_state(sock):
    # TCP_INFO 值为 11，返回一个 struct tcp_info
    info = sock.getsockopt(socket.IPPROTO_TCP, 11, 128)
    # 偏移量 1 处为 tcpi_state，不同值对应不同状态（1=ESTABLISHED）
    state = struct.unpack('B', info[0:1])[0]  # 简化：真实需解析整个结构
    return state

# 此时双方均为 ESTABLISHED，说明三次握手完成
print('客户端状态:', tcp_state(client))
print('服务端状态:', tcp_state(conn))

# 发送数据验证双向传输
client.send(b'hello')
data = conn.recv(1024)
print('收到数据:', data)

# 关闭连接，触发四次挥手
client.close()
conn.close()
server.close()

# 说明：上述代码中，三次握手发生在 connect() 调用内部，由内核自动完成，用户态无法直接控制每一步。若要观察 SYN 包，可使用 tcpdump 或 scapy 构造原始报文。

# SYN Cookie 验证伪代码（内核层）:
# 服务端 listen 时，若设置 /proc/sys/net/ipv4/tcp_syncookies=1，当半连接队列满时，内核不分配 TCB，而是计算 cookie 并放入 SYN-ACK 的 seq 字段。
# 客户端收到 SYN-ACK 后，回复 ACK(seq=cookie+1)。服务端校验 cookie 有效性，若通过则直接建立连接，否则丢弃。
# 可用 scapy 模拟：发送 SYN，收到 SYN-ACK 中 seq 即 cookie；修改 cookie 后发送 ACK，服务端不会建立连接（RST 或忽略）。
# 实际验证命令：
# sysctl -w net.ipv4.tcp_syncookies=1
# tcpdump -i lo 'tcp[tcpflags] & (tcp-syn|tcp-ack) != 0' -n  # 抓包观察握手包
```

### 4. 常见误区与进阶思考
误区1：认为三次握手是“三次数据传输”。实际上握手包中不携带应用数据（严格说第三次 ACK 可携带数据，但大多数实现不这么做），其目的是同步序列号，而非传输数据。很多工程师误以为连接建立后立即可以发送数据，却忽略 TCP 的慢启动和拥塞窗口初始值。误区2：认为 SYN Cookie 与正常握手是二选一的机制。实际上 SYN Cookie 是防御性策略，仅在半连接队列满时启用（或 sysctl 设置为 always 时开启），它会导致某些 TCP 扩展（如时间戳、窗口缩放）失效，因为 cookie 空间有限，无法编码全部参数。因此需要权衡安全性与功能完整性。

思考题：假设一个客户端发送 SYN 后，服务端回复 SYN-ACK，但该 SYN-ACK 在网络中丢失，客户端在超时后重发 SYN。此时服务端是重新分配一个新的 cookie 还是复用之前生成的 cookie？如果复用，会有什么安全问题？请结合 SYN Cookie 的哈希密钥和时效性分析。提示：内核通常使用当前时间戳作为哈希因子之一，若重传发生在同一秒内，cookie 可能相同；但若跨秒，则 cookie 不同。这可能导致客户端在重传时收到两个不同的 SYN-ACK，最终客户端只响应其中一个 ACK，而另一个连接则被服务端忽略。这个设计如何防止重放攻击？请深入思考。
