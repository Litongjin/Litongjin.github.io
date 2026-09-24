---
title: "每日基础技术总结 · 2026-09-09 · TCP三次握手与四次挥手中SYN/ACK包的序列号初始值及状态迁移细节"
date: 2026-09-09 08:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-09 · TCP三次握手与四次挥手中SYN/ACK包的序列号初始值及状态迁移细节

## 📚 今日主题

> **TCP三次握手与四次挥手中SYN/ACK包的序列号初始值及状态迁移细节**（网络基础）

### 1. 核心概念速览
TCP三次握手与四次挥手是传输层可靠连接建立与释放的核心状态机。其本质是通信双方通过交换SYN、ACK、FIN报文，对各自初始序列号（ISN）进行同步并确认对方序列号，从而为双向字节流建立一个可靠的全局上下文。SYN和FIN各消耗一个序列号，ACK不消耗序列号。该机制位于传输层，是IP之上、应用层之下，所有可靠传输逻辑的根基。专业工程师必须掌握，因为连接池、负载均衡、代理、容器网络、云原生架构中的连接行为都以此为底层约束。

### 2. 底层原理剖析
三次握手的过程如下：
1. 客户端发送 SYN(seq=x)，客户端状态 CLOSED->SYN_SENT。
2. 服务端收到后返回 SYN+ACK(seq=y, ack=x+1)，服务端状态 LISTEN->SYN_RCVD。
3. 客户端收到后发送 ACK(seq=x+1, ack=y+1)，客户端状态 SYN_SENT->ESTABLISHED；服务端收到该ACK后状态 SYN_RCVD->ESTABLISHED。
其中 x 与 y 是双方独立随机生成的初始序列号。因为 SYN 报文会消耗一个序列号，所以确认号是对方ISN+1；后续第一个数据字节的序列号分别为 x+1 和 y+1。

四次挥手的过程如下（以客户端主动关闭为例）：
1. 客户端发送 FIN(seq=u)，客户端 ESTABLISHED->FIN_WAIT_1。其中 u 等于客户端最后一次发送的数据序列号+1。
2. 服务端回复 ACK(ack=u+1)，服务端 ESTABLISHED->CLOSE_WAIT，客户端 FIN_WAIT_1->FIN_WAIT_2。此时半关闭，服务端仍可向客户端发送数据。
3. 服务端应用层关闭时发送 FIN(seq=v)，服务端 CLOSE_WAIT->LAST_ACK。v 是服务端最后一次发送的数据序列号+1。
4. 客户端回复 ACK(ack=v+1)，客户端 FIN_WAIT_2->TIME_WAIT，服务端收到后 LAST_ACK->CLOSED。客户端等待2MSL后自动进入CLOSED。
ACK 不消耗序列号，因此服务端在步骤2中发送ACK时，自己的发送序列号 v 保持不变。

对比前端已有的 Promise 状态机：二者都是有限状态机，但 Promise 的迁移方向是单向的、只迁移一次，由异步结果驱动；TCP 状态机是双向的、可由多个报文事件驱动，并伴随超时重传、半关闭等中间状态，复杂性远高于应用层的状态模型。

### 3. 基础代码与实战验证
```text
# 伪代码：模拟三次握手与四次挥手中序号/确认号变化及状态迁移

# 双方初始状态
client_state = CLOSED
server_state = CLOSED

# 双方独立生成初始序列号
client_isn = random()
server_isn = random()

# ---------- 三次握手 ----------
# 第1步：C -> S  SYN(seq=client_isn)
send_packet(dst=S, type=SYN, seq=client_isn)
client_state = SYN_SENT

# 第2步：S -> C  SYN+ACK(seq=server_isn, ack=client_isn+1)
# SYN 消耗一个序号，所以 ack = client_isn + 1
send_packet(dst=C, type=SYN+ACK, seq=server_isn, ack=client_isn+1)
server_state = SYN_RCVD

# 第3步：C -> S  ACK(seq=client_isn+1, ack=server_isn+1)
# 客户端发出的第一个数据字节序号是 client_isn+1
send_packet(dst=S, type=ACK, seq=client_isn+1, ack=server_isn+1)
client_state = ESTABLISHED
server_state = ESTABLISHED

# ---------- 数据阶段（假设双方发送了若干字节）----------
client_last_seq = client_isn + 1 + client_data_len
server_last_seq = server_isn + 1 + server_data_len

# ---------- 四次挥手（客户端主动关闭）----------
# 第1步：C -> S  FIN(seq=client_last_seq)
send_packet(dst=S, type=FIN, seq=client_last_seq)
client_state = FIN_WAIT_1

# 第2步：S -> C  ACK(ack=client_last_seq+1)
# FIN 消耗一个序号；ACK 本身不消耗序号，所以服务端发送序列号仍是 server_last_seq
send_packet(dst=C, type=ACK, seq=server_last_seq, ack=client_last_seq+1)
server_state = CLOSE_WAIT
client_state = FIN_WAIT_2

# 第3步：S -> C  FIN(seq=server_last_seq)
send_packet(dst=C, type=FIN, seq=server_last_seq)
server_state = LAST_ACK
client_state = TIME_WAIT

# 第4步：C -> S  ACK(ack=server_last_seq+1)
send_packet(dst=S, type=ACK, seq=client_last_seq+1, ack=server_last_seq+1)
server_state = CLOSED

# 客户端经过 2MSL 后自动进入 CLOSED
```

### 4. 常见误区与进阶思考
误区1：认为ACK包也消耗一个序列号。实际上只有SYN和FIN这种标识连接状态变化的控制报文消耗序号，ACK只是一个确认信息，不占用序号空间。若混淆这一点，后续计算重传序列号、滑动窗口可用空间时会直接出错。
误区2：认为四次挥手时，被动关闭方的ACK与FIN总是合并发送。现实中被动方在收到FIN后可能仍有数据未发送完毕，因此ACK与FIN往往分开发送，导致CLOSE_WAIT和FIN_WAIT_2这两个中间状态可能持续较长时间。这也是排查大量TCP连接堆积时必须关注的状态。

思考题：在三次握手中，如果客户端发出的第三个ACK丢失，服务端会一直处于SYN_RCVD并超时重传SYN+ACK，此时客户端已经处于ESTABLISHED。当客户端收到重传的SYN+ACK后，会如何响应？这一行为如何体现TCP对丢包与重复报文的处理机制？
