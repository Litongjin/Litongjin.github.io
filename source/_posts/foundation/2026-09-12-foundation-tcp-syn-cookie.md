---
title: "每日基础技术总结 · 2026-09-12 · TCP 三次握手与 SYN Cookie"
date: 2026-09-12 08:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-12 · TCP 三次握手与 SYN Cookie

## 📚 今日主题

> **TCP 三次握手与 SYN Cookie**（网络基础）

### 1. 核心概念速览
TCP 三次握手是传输控制协议在建立连接时交换 SYN、SYN-ACK、ACK 三个控制报文，以完成初始序列号（ISN）双向同步、协商传输参数并分配传输控制块（TCB）的握手协议。其本质不是简单的“客户端请求、服务端应答”，而是把不可靠的 IP 层抽象为可靠字节流通信前的状态同步过程。SYN Cookie 是一种无状态抗 SYN Flood 机制：当半连接队列溢出或系统开启该防护时，服务器不保存请求端的半连接记录，而是将连接四元组、时间窗口、MSS 等数据经密码学散列编码为 SYN-ACK 报文中的初始序列号（即 cookie），收到客户端的第三次 ACK 时再反向校验并重建连接状态。

它在 TCP/IP 协议栈传输层，是 TCP 连接生命周期的第一阶段；后端的半连接管理、连接超时、L4 负载均衡、DDoS 防护完全依赖对它精确的底层认知。专业工程师必须掌握它，否则抓包时只能看到表面报文，无法定位连接失败、资源耗尽、NAT 或丢包后的状态分歧。AI 系统上层无论跑 HTTP/gRPC 还是分布式数据流，底层可靠传输仍建立在这 3 个报文之上。

### 2. 底层原理剖析
1. 状态机与报文语义
客户端从 CLOSED 进入 SYN_SENT；服务器从 LISTEN 进入 SYN_RCVD；客户端收到 SYN-ACK 后进入 ESTABLISHED；服务器收到最终的 ACK 后进入 ESTABLISHED。

报文序列：
  Client -> Server: SYN, seq = x
  Server -> Client: SYN|ACK, seq = y, ack = x + 1
  Client -> Server: ACK, seq = x + 1, ack = y + 1

为什么必须三次：第一次 SYN 告知服务器存在连接意图；第二次 SYN-ACK 让客户端确认服务器收到了自己的同步请求，同时拿到服务器的 ISN；第三次 ACK 让服务器确认客户端接收到了自己的 SYN-ACK。如果只有两次，服务器无法区分客户端是否已同步自己的 ISN，延迟重传的旧 SYN 也可能令服务器误建新连接而浪费 TCB；三次握手本质是一次双向 ISN 同步和确认。

2. SYN Cookie 的底层机制
Linux 内核 net/ipv4/syncookies.c 是典型实现。当 tcp_syncookies=1 且系统半连接资源紧张时，收到 SYN 后不创建 request_sock，而是计算：
  cookie = HASH(源IP, 源端口, 目的IP, 目的端口, 时间戳低位, 密钥) | MSS 索引
并把该 cookie 作为 SYN-ACK 的序列号发出。客户端正常回复 ACK 时，其 ack 字段等于 cookie + 1。服务器收到 ACK 后，用同样的四元组、时间窗和密钥重新计算 cookie；若 ack - 1 与 cookie 匹配，便认定这是合法的第三次握手，此时才分配 TCB 并进入 ESTABLISHED。由此避免恶意 SYN 流填满内核半连接表。

3. 与前端已有概念的异同
三次握手更像运行期协议协商，而不是 Java interface 与 TS interface 那种编译期/运行期类型契约。Java interface 在运行时通过多态分派强制方法约定；TS interface 是结构化类型检查，编译后擦除。TCP 握手没有静态类型系统，它用带内报文和状态机来同步“双方能否可靠互发数据”这一事实。你可以把它类比成前后端之间的一次能力协商：前端的 TypeScript 类型只在开发期生效，而 TCP 握手在每次真实连接建立时都要重新执行。与 HTTP 请求-响应线性模型的关键差异：TCP 是双向序列号和确认号的持续状态耦合；即便握手完成，后续每个字节流都受这个初始状态约束。

### 3. 基础代码与实战验证
```text
下面用 Node.js 原生 net 模块构造最简 TCP 回声服务器；SYN Cookie 部分给出内核算法精炼伪代码。

// server.js：最简 TCP 服务器，无第三方依赖
const net = require('net');

const server = net.createServer((socket) => {
  // connection 事件触发意味着内核已完成三次握手，socket 处于 ESTABLISHED
  // 用户态只看到连接可用，看不到 SYN/SYN-ACK/ACK 报文；这些全部由内核协议栈完成
  socket.pipe(socket); // 接收到什么就原样写回，验证 TCP 双向字节流
});

server.listen(8080, '0.0.0.0', () => {
  console.log('TCP server listening on 8080');
});

// 抓包验证：
// tcpdump -i any port 8080 -nn 'tcp[tcpflags] & (tcp-syn|tcp-ack) != 0'
// 执行 curl http://127.0.0.1:8080/ 后，可看到如下顺序：
// 1) SYN seq=x
// 2) SYN-ACK seq=y ack=x+1
// 3) ACK seq=x+1 ack=y+1

// SYN Cookie 校验伪代码（内核实测逻辑）
function handleThirdAck(ackPacket) {
  const expectedCookie = hash(
    ackPacket.srcIp,
    ackPacket.srcPort,
    ackPacket.dstIp,
    ackPacket.dstPort,
    currentTimeBucket(),
    secretKey
  );
  // ack 字段表示期望收到的下一字节；握手里的 ack = cookie + 1
  if (ackPacket.ack - 1 === expectedCookie) {
    // cookie 校验通过，这一刻才创建传输控制块并进入 ESTABLISHED
    createConnection(ackPacket);
  } else {
    // 无状态丢弃，不分配任何内核内存
    dropPacket();
  }
}

// 压力验证命令：
// sysctl -w net.ipv4.tcp_syncookies=1
// sysctl -w net.ipv4.tcp_max_syn_backlog=1024
// 发起大量仅 SYN 的流量后，用 netstat -s | grep syncookie
// 观察 TcpExtSyncookiesSent 和 TcpExtSyncookiesRecv 增量，即为 SYN Cookie 生效证据。
```

### 4. 常见误区与进阶思考
1. 误区：认为三次握手是客户端发起、服务端同意的准实时请求。实际客户端发出最后一个 ACK 后立即进入 ESTABLISHED，服务器收到 ACK 前仍处于 SYN_RCVD。如果最后一个 ACK 丢失，服务器会重传 SYN-ACK，客户端却已认为连接可用并开始发数据；虽然客户端发来的数据段中 ACK 标志也能间接完成第三次握手，但在此之前服务器没有主动推送数据的资格，两端可见状态存在时间差。真实协议栈里没有“双端同时 ESTABLISHED”的原子时刻。
2. 误区：把 SYN Cookie 当作零成本且最安全的防护。它确实省下半连接队列，但代价是：通常只能编码少量传输选项（如 MSS），无法完整协商窗口缩放、时间戳、SACK；每次校验都需要 CPU 和密钥，高速随机 ACK 也会消耗计算资源；它只解决半连接资源耗尽，不解决应用层合法请求耗尽，更不能替代防火墙和速率限制。生产环境必须把它理解为“保命兜底”，而不是默认最优解。

进阶思考题：
如果服务器收到一个纯 ACK 报文，该四元组在 TCP 控制块哈希表中不存在，且其 ack 字段恰好等于某个旧握手 cookie + 1，服务器应如何处理？它与开启 SYN Cookie 时收到第三次 ACK 的处理路径有何本质区别？请从 tcp_v4_do_rcv 的 sock 查找顺序和 request_sock 的生成时机回答。
