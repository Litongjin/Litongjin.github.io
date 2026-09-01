---
title: "每日基础技术总结 · 2026-09-01 · TCP 三次握手与 SYN Cookie"
date: 2026-09-01 08:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-01 · TCP 三次握手与 SYN Cookie

## 📚 今日主题

> **TCP 三次握手与 SYN Cookie**（网络基础）

### 1. 核心概念速览
TCP 三次握手是传输控制协议（TCP）在面向连接通信中建立可靠逻辑连接的核心机制。其本质是通信双方通过交换三个报文段，完成序列号（ISN）的同步与双方收发能力的确认，确立双向数据传输的初始状态。三次握手解决的核心问题是：在网络不可靠、存在延迟和重复报文的条件下，如何避免历史重复连接请求造成的数据混乱，并确保双方都清楚对方已准备好收发数据。SYN Cookie 是防御 SYN Flood 攻击的一种无状态握手优化机制，其本质是将连接请求状态编码进服务端返回的序列号中，使服务端在握手完成前不分配任何连接资源，从而抵抗大量伪造源地址的握手请求。在整个计算机体系中，TCP 握手位于 L4 传输层，是所有可靠网络应用的基石；专业工程师必须理解其状态机、报文语义及资源管理，才能在性能调优、网络故障排查和抗 DDoS 设计时做出正确决策。

### 2. 底层原理剖析
三次握手的底层机制可精确描述为状态机迁移与报文交换过程：

1. CLOSED -> SYN_SENT：主动方 A 生成初始序列号 ISN_A，发送 SYN=1, seq=ISN_A 的报文段，进入 SYN_SENT。
2. SYN_SENT -> ESTABLISHED：被动方 B 收到 SYN 后，若接受连接，则分配传输控制块（TCB，包含收发缓冲区、序列号状态等资源），生成自身初始序列号 ISN_B，发送 SYN=1, ACK=1, seq=ISN_B, ack=ISN_A+1，进入 SYN_RCVD。
3. ESTABLISHED：A 收到 SYN+ACK 后，确认 B 的收发能力，发送 ACK=1, seq=ISN_A+1, ack=ISN_B+1。B 收到该 ACK 后，确认 A 的接收能力，双方进入 ESTABLISHED，连接建立。

关键本质：第三次 ACK 的作用不仅是确认，更重要的是防止已失效的 SYN 请求在网络上滞留后到达 B，导致 B 建立一条 A 已废弃的连接。若采用两次握手，A 发出的旧 SYN 到 B 后，B 会将已过期的连接当作新连接接受，从而浪费资源并可能造成数据错误。三次握手通过 A 对 B 的 SYN+ACK 进行最终确认，使 B 能辨别是否为当前活跃请求。

SYN Cookie 机制在三次握手上的变体如下：
- 当服务端收到 SYN 时，不分配 TCB，而是根据客户端 IP、端口、服务端 IP、端口以及一个密钥，通过哈希函数生成一个 32 位整数的 cookie，将其作为 ISN_B 放入 SYN+ACK 回应。
- 服务端完全不需要在本地保存半连接信息，即无状态分配。
- 客户端收到 SYN+ACK 后，依然回复 ACK，其中 ack = ISN_B + 1。设置在这个 ACK 中的确认号（ack）就等于 cookie+1。
- 服务端收到 ACK 时，重新计算 cookie，并验证 ack-1 是否等于当前计算出的 cookie。若相等，则根据 cookie 中的编码信息（如 MSS 等）重建连接参数，分配 TCB，进入 ESTABLISHED。
这种机制将连接的合法性验证后移，资源分配从 SYN 到达时推迟到 ACK 验证成功后，有效抵御不发送最终 ACK 的 SYN Flood。

与前端已有概念的对比：前端开发中常见的“接口”概念（如 TypeScript 接口、Java Interface）描述的是“契约”或“类型形状”，它定义的是静态约定；而 TCP 握手定义的是“状态同步协议”。TS 接口在编译期就能静态检查，不涉及运行时资源分配；TCP 握手则是运行时通过报文交换驱动状态机，涉及资源生命周期。两者的相似点在于都是一种约定：TS 接口约定了对象结构，TCP 握手约定了连接双方必须遵守的序列号和确认号规则。但 TS 接口没有任何状态机概念，而 TCP 握手是一个严格状态迁移过程。另外，前端常说的“回调”（callback）与 TCP 的“确认”（ack）也有类比但本质不同：回调是异步完成后的动作，TCP ack 是协议层面确认数据已收到的控制信息。理解这些差异有助于工程师从应用层思维转向传输层思维。

### 3. 基础代码与实战验证
```text
纯代码难以在用户态模拟内核协议栈行为，但可以用 sockets API 的调用序列精确展示三次握手的用户在态映射。以下用 Node.js 的 net 模块展示服务端监听与客户端连接时，内核自动完成握手的代码再，关键事件对应底层报文交换：

// server.js
const net = require('net');
// 调用 listen 后，内核进入 LISTEN 状态，维护 accept 队列
const server = net.createServer(socket => {
  // 当握手完成，连接进入 ESTABLISHED，内核将连接放入完成队列，事件循环回调此函数
  console.log('connection established, remote address:', socket.remoteAddress);
  socket.on('data', d => console.log('data received:', d.toString()));
});
server.listen(8080, '127.0.0.1');

// client.js
const net = require('net');
// 调用 connect 时，内核发送 SYN，状态 SYN_SENT
const client = net.connect(8080, '127.0.0.1', () => {
  // 回调触发时，客户端内核已收到 SYN+ACK 并发出 ACK，状态为 ESTABLISHED
  console.log('client connected');
  client.write('hello');
});

// 底层报文顺序（用 tcpdump 验证）：
// 1. IP.src=client:port > IP.dst=server:8080: Flags [S], seq=C
// 2. IP.src=server:8080 > IP.dst=client:port: Flags [S.], seq=S, ack=C+1
// 3. IP.src=client:port > IP.dst=server:8080: Flags [.], ack=S+1

// 验证 SYN Cookie 的伪代码（Linux 内核简化逻辑）：

function syn_cookie(src_ip, src_port, dst_ip, dst_port, secret) {
  // 用哈希函数将四元组和密钥压缩到 24 位，再拼接 3 位时间计数和 5 位 MSS 索引
  const h = hash(src_ip, src_port, dst_ip, dst_port, secret);
  const counter = time_counter / 64;  // 每 64 秒递增
  const mss_idx = encode_mss(mss);
  return (counter << 26) | (h & 0x00FFFFFF) | (mss_idx << 16) & 0xFFFF; // 实际布局因内核版本而异
}

// 收到 ACK 后的验证：
function check_cookie(ack_value, src_ip, src_port, dst_ip, dst_port, secret) {
  const cookie = ack_value - 1;
  const counter = cookie >> 26;
  if (abs(counter - current_counter) > 3) return false; // 防止重放，检查时间窗
  const mss_idx = (cookie >> 16) & 0x07;
  if (mss_idx >= mss_table.length) return false;
  // 重新计算 hash 并比对
  const h = hash(src_ip, src_port, dst_ip, dst_port, secret);
  return (cookie & 0x00FFFFFF) === (h & 0x00FFFFFF);
}

// 上述伪代码展示服务端在收到 ACK 前不存任何连接状态，仅通过 ISN 携带信息完成校验。
```

### 4. 常见误区与进阶思考
误区 1：认为三次握手是“确认双方都有发送和接收能力”。这在逻辑上看起来合理，但本质是“避免历史重复 SYN 造成的错误连接”。TCP 的可靠传输保证序列号不被错误地复用，三次握手确保连接建立时的初始序列号不被旧报文干扰。若仅确认能力，两次握手已足够，但无法防止旧 SYN 延迟到达而建立无效连接。工程师常忽略 ack 号 +1 的含义：它代表“你要发送的下一个字节的序列号”，这正是同步双方的发送窗口起点。

误区 2：认为 SYN Cookie 只在攻击时才开启，且不影响正常连接。实际上，若 SYN Cookie 被默认开启，它会牺牲部分 TCP 扩展功能（如时间戳、大窗口缩放等），因为 cookie 能编码的信息有限；且服务端在完成三次握手前无法获知客户端是否真正可达，也无法为合法连接预分配缓冲区，可能导致性能下降。正确做法是在 SYN 队列积压超过阈值时动态启用，或通过现代内核的 SYN Proxy 机制结合 Cookie 与状态化队列。

思考题：如果客户端在 TCP 连接建立后立即收到一个携带旧序列号的重复 SYN+ACK 报文，按三次握手机制该报文会被客户端忽略吗？如果不会忽略，协议栈如何通过序列号与状态机判定其为无效报文？请结合 TCP 状态迁移与序列号验证机制回答。
