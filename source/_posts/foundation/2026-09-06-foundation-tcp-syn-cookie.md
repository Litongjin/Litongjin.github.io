---
title: "每日基础技术总结 · 2026-09-06 · TCP 三次握手与 SYN Cookie"
date: 2026-09-06 08:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-06 · TCP 三次握手与 SYN Cookie

## 📚 今日主题

> **TCP 三次握手与 SYN Cookie**（网络基础）

### 1. 核心概念速览
TCP 三次握手是传输控制协议（TCP）在连接建立阶段使用的同步机制，本质是通过交换初始序列号（ISN）与确认号，在通信双方之间确立双向的可靠字节流状态。它解决的核心问题是：在不可靠的IP网络之上，如何使两端就初始序列号、窗口大小、最大报文段长度等连接参数达成一致，并同步各自的发送/接收窗口状态。三次握手的必要性在于，TCP是全双工协议，且报文可能延迟、重排或重复，因此需要双方各自证明自己能够接收和发送数据。该机制位于传输层（OSI第4层），是TCP状态机中从CLOSED到ESTABLISHED的路径，也是可靠传输、流量控制、拥塞控制的前提。专业工程师必须掌握它，因为它直接关系到连接性能、安全漏洞（如SYN Flood）及中间设备（NAT、防火墙）行为，是排查网络故障和设计高并发服务的基础。SYN Cookie是一种针对SYN Flood攻击的无状态防御机制，其本质是服务器在收到SYN时不分配任何连接资源，而是通过构造一个基于连接四元组（源IP、源端口、目的IP、目的端口）、ISN、时间戳及密钥的哈希值作为SYN-ACK的序列号（即Cookie），仅在收到客户端ACK时验证Cookie有效后，再重建连接控制块。它将三次握手中的资源分配从第一次握手延迟到第三次握手，从而避免半连接队列被恶意请求占满。

### 2. 底层原理剖析
三次握手的底层逻辑如下：
1. 客户端发送SYN报文，其中序列号seq=x（由客户端随机生成，称为ISN_c）。此时客户端进入SYN_SENT状态。该报文不携带应用数据，但占用一个序列号。
2. 服务器收到SYN后，必须同时确认客户端的ISN并发出自己的ISN。因此发送SYN-ACK报文，其中ack=x+1（表示期望收到x+1），seq=y（ISN_s）。服务器进入SYN_RCVD状态，并分配传输控制块（TCB），将该连接放入半连接队列。
3. 客户端收到SYN-ACK后，发送ACK报文，其中seq=x+1，ack=y+1。客户端进入ESTABLISHED状态；服务器收到ACK后也进入ESTABLISHED状态，并将连接从半连接队列移入全连接队列。

伪代码描述：
```
Client: send(SYN, seq=x) -> SYN_SENT
Server: recv(SYN) -> allocate TCB; enqueue half-open; send(SYN+ACK, seq=y, ack=x+1) -> SYN_RCVD
Client: recv(SYN+ACK) -> verify ack==x+1; send(ACK, seq=x+1, ack=y+1) -> ESTABLISHED
Server: recv(ACK) -> verify ack==y+1; dequeue half-open; move to established; -> ESTABLISHED
```

SYN Cookie机制原理：
服务器收到SYN时不分配TCB，而是计算：
- cookie = hash(源IP, 源端口, 目的IP, 目的端口, 密钥, 计数器/时间戳) 的低24位（或更多位），其中也隐含了MSS编码。
将cookie作为SYN-ACK的初始序列号y。服务器丢弃该SYN的请求记录（不维护半连接）。
当客户端返回ACK时，其ack字段为y+1。服务器提取cookie值（ack-1），重新hash当前四元组和时间戳，若匹配且时间戳在有效窗口内，则按cookie中的编码恢复MSS等参数，直接创建TCB进入ESTABLISHED。

与前端已有概念的对比：
- 类似React的虚拟DOM reconciliation：三次握手是“协商-确认-确认确认”，虚拟DOM也是先渲染生成新的状态，再通过diff确认差异后提交更新。但TCP是双向确认，而虚拟DOM是单向diff。
- 类似JavaScript的Promise状态机：三次握手中的状态转换（SYN_SENT, SYN_RCVD, ESTABLISHED）类似Promise的pending/fulfilled状态转移，但TCP状态转移更严格且依赖网络往返。
- 类似TypeScript的类型系统：类型检查在编译期（握手阶段）建立契约，运行时（传输阶段）不必再验证；SYN Cookie则类似“类型断言”加“运行时校验”，前期不建立完整类型检查，等到真正收到ACK（类型验证）时才建立连接。
- 更贴近的类比：前端中HTTP/2的流复用和TCP连接池。但本质区别在于TCP是字节流，不是消息边界。

### 3. 基础代码与实战验证
真实环境可用Python标准库socket和struct模拟TCP握手（仅演示，实际底层由内核完成）。以下代码展示服务器监听并打印客户端连接，以及客户端连接的时序。

```python
import socket
import threading

# 服务器端：模拟三次握手的接收过程（内核自动完成，此处仅观察状态）
def server():
    srv = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    srv.bind(('127.0.0.1', 9999))
    srv.listen(1)  # listen() 的backlog表示全连接队列大小，内核半连接队列由tcp_max_syn_backlog控制
    conn, addr = srv.accept()  # accept() 从全连接队列取出一个已三次握手完成的连接
    print(f"[Server] established: {addr}")
    conn.close()

# 客户端：connect() 触发内核发送SYN并完成握手
def client():
    cli = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    cli.connect(('127.0.0.1', 9999))  # connect() 返回时，说明三次握手已完成（或失败）
    print("[Client] connected")
    cli.close()

if __name__ == "__main__":
    threading.Thread(target=server).start()
    threading.Thread(target=client).start()
```

关键行注释：
- `srv.listen(1)`：内核为该socket创建半连接队列和全连接队列。backlog指定全连接队列容量，并非最大连接数。半连接队列长度由系统参数`net.ipv4.tcp_max_syn_backlog`控制。
- `srv.accept()`：系统调用从全连接队列中摘取一个已完成握手的连接，返回一个新的socket描述符；若队列为空则阻塞。
- `cli.connect()`：系统调用内部发起三次握手，同步阻塞直到连接状态转为ESTABLISHED或失败（如超时、RST）。

若要观察真实握手状态，可在Linux下使用`tcpdump -i lo port 9999`抓包查看SYN、SYN-ACK、ACK三个报文；或用`ss -tn`查看socket状态。

伪代码展示SYN Cookie核心逻辑：
```
// 服务器收到SYN
function handle_syn(pkt):
    if syn_cookie_enabled and half_open_queue_full:
        cookie = hash(pkt.src, pkt.dst, secret, counter) & 0x00FFFFFF
        mss_encoded = encode_mss(pkt.mss) << 24  // 高位编码MSS
        seq = cookie | mss_encoded
        send(SYN+ACK, seq=seq, ack=pkt.seq+1)
        // 不分配TCB，不加入半连接队列
    else:
        // 正常三次握手

// 服务器收到ACK
function handle_ack(pkt):
    if pkt.ack == 0:
        return
    received_cookie = pkt.ack - 1
    expected_cookie = hash(pkt.src, pkt.dst, secret, counter) & 0x00FFFFFF
    if (received_cookie & 0x00FFFFFF) == expected_cookie:
        mss = decode_mss(received_cookie >> 24)
        // 重建TCB，建立连接
        create_connection(pkt)
    else:
        // 丢弃或RST
```

### 4. 常见误区与进阶思考
误区一：认为三次握手中第一次SYN和第二次SYN-ACK都包含实际数据。实际上，TCP序列号空间中的序列号在握手阶段是被占用的，但SYN和SYN-ACK本身不携带数据；只有第三次ACK可以携带数据（某些实现允许），但通常不携带。因此，三次握手纯粹是控制面同步，数据面从握手完成后才开始。
误区二：将SYN Cookie视为解决所有SYN Flood的银弹。SYN Cookie会损失部分TCP特性（如窗口缩放、时间戳、SACK等无法在握手时协商），且增加服务器CPU计算哈希的负担。大型生产环境通常采用混合策略：半连接队列满时才启用SYN Cookie，而不是默认开启。另外，SYN Cookie无法防御所有DDoS，它只缓解半连接队列耗尽，对全连接队列溢出或应用层DDoS无能为力。

思考题：在TCP三次握手中，如果客户端发送的SYN丢失，客户端会重发SYN；如果服务器的SYN-ACK丢失，客户端会重发SYN，但服务器已经进入SYN_RCVD并发送了SYN-ACK。此时客户端重发的SYN到达服务器，服务器会如何处理？请结合TCP状态机和序列号设计，解释服务器是否会直接重发SYN-ACK，还是会重新处理SYN？这揭示了TCP如何处理重复报文与同步异常，是理解可靠传输的关键。
