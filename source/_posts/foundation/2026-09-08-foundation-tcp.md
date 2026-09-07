---
title: "每日基础技术总结 · 2026-09-08 · TCP 三次握手 / 四次挥手"
date: 2026-09-08 07:13:39
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-08 · TCP 三次握手 / 四次挥手

## 📚 今日主题

> **TCP 三次握手 / 四次挥手**（前端底层与计算机基础）

### 1. 核心概念速览
TCP三次握手/四次挥手是TCP协议用于建立和终止可靠连接的机制。三次握手本质上是在不可靠IP网络上双向同步初始序列号（ISN）并确认双方收发能力，从而建立一条有序、可靠的全双工逻辑信道；四次挥手则是因TCP全双工特性，每个方向的数据发送必须独立关闭，故需要四次交互来完成双方数据的完整交付与确认。该机制位于传输层，是所有面向连接的应用（HTTP、HTTPS、WebSocket等）的可靠性基石。专业工程师掌握它，是为了准确理解连接建立开销、进行性能调优（连接复用、重传超时、拥塞控制）以及高效排查网络故障（握手超时、半连接、TIME_WAIT堆积等）。

### 2. 底层原理剖析
三次握手过程：
1. 客户端发送SYN段，seq=x，进入SYN_SENT状态。
2. 服务端回复SYN+ACK段，seq=y，ack=x+1，进入SYN_RCVD状态。
3. 客户端发送ACK段，seq=x+1，ack=y+1，双方进入ESTABLISHED状态。

本质：双向同步序列号。第一次握手使服务端确认客户端发送能力；第二次握手使客户端确认服务端收发能力，同时服务端也确认了客户端发送能力；第三次握手使服务端确认客户端接收能力（因为客户端已收到服务端的SYN，但需让其确认）。如果只有两次，则服务端无法确认客户端是否能收到自己的SYN，且可能接受历史遗留的旧SYN请求，导致资源浪费。

四次挥手过程（以主动关闭方A，被动关闭方B为例）：
1. A发送FIN段，seq=m，进入FIN_WAIT_1。
2. B回复ACK段，ack=m+1，进入CLOSE_WAIT；A收到后进入FIN_WAIT_2。
3. B应用层完成关闭后发送FIN段，seq=n，进入LAST_ACK。
4. A回复ACK段，ack=n+1，进入TIME_WAIT，等待2MSL后关闭；B收到ACK后立即CLOSED。

与前端已有概念对比：前端中的'接口'在TypeScript与Java中有本质差异——TS接口是结构类型（structurally typed），编译期检查，无运行时代价；Java接口是名义类型（nominally typed），且可作为运行时多态契约。而TCP握手是传输层的运行时协议状态机，两者的'契约'语义发生在不同层次，但思想相通：都是通过明确双方可接受的能力集合来建立信任。前端熟悉的HTTP在TCP之上，一次HTTP请求通常复用一个TCP连接（keep-alive），因此三次握手的开销只发生一次；HTTP/2和HTTP/3进一步通过多路复用或QUIC（基于UDP）来降低连接建立成本，理解TCP握手是分析这些协议优化的前提。

### 3. 基础代码与实战验证
```text
以下以Python socket为例，展示三次握手与四次挥手的用户态API映射（实际握手/挥手由Linux内核完成）：

服务端：
import socket
server = socket.socket(socket.AF_INET, socket.SOCK_STREAM) # 创建TCP套接字
server.bind(('127.0.0.1', 8080)) # 绑定地址
server.listen(5) # 进入LISTEN状态，内核维护半连接队列和全连接队列
conn, addr = server.accept() # 阻塞直至三次握手完成；从全连接队列取出已建立连接
# 此时握手已完成，conn对应一个新的已连接套接字，seq/ack已同步

客户端：
import socket
client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
client.connect(('127.0.0.1', 8080)) # 内核发送SYN，收到SYN+ACK后回ACK，然后返回；返回即握手成功

数据收发：
client.send(b'hello') # 数据按序发送，内核负责重传、排序
client.recv(1024) # 阻塞接收

四次挥手：
conn.close() # 主动关闭，内核发送FIN，进入FIN_WAIT_1
# 对端recv返回b''，应用层可感知EOF，随后对端调用close发送FIN
# 主动关闭端收到FIN后回ACK，进入TIME_WAIT，等待2MSL后彻底关闭

注意：用户态看到的只是connect/accept/close返回，真正的握手报文在系统调用过程中由内核协议栈自动生成。
```

### 4. 常见误区与进阶思考
常见误区：
1. 认为三次握手只是'客户端和服务端互相打招呼'，忽略了核心是双向序列号同步和防止历史连接请求复用。如果没有第三次握手，服务端无法区分新SYN和网络重传的旧SYN，可能建立错误连接并浪费资源。
2. 认为四次挥手一定是客户端先发起、服务端被动响应，或者认为一定恰好是四次报文。实际上谁先调用close谁就是主动关闭方；被动方的ACK和FIN分别由内核自动回应和应用层close触发，可能合并也可能分离，因此抓包常见的是三个或四个报文段。

进阶思考题：
主动关闭方发出最后一个ACK后，为什么不立即进入CLOSED，而是进入TIME_WAIT并等待2MSL？请从ACK丢失重传和防止旧报文段污染新连接两个角度，推导TIME_WAIT的必要性。
