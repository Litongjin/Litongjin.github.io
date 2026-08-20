---
title: "每日基础技术总结 · 2026-08-16 · TCP 的 SYN 洪水攻击与防御"
date: 2026-08-16 08:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-08-16 · TCP 的 SYN 洪水攻击与防御

## 📚 今日主题

> **TCP 的 SYN 洪水攻击与防御**（网络基础）

### 1. 核心概念速览
TCP SYN 洪水攻击（SYN Flood）是一种针对 TCP 三次握手协议的拒绝服务（DoS）攻击。攻击者向目标服务器发送大量伪造源 IP 的 SYN 报文，使服务器为每个半连接分配内核资源（TCB，传输控制块）并进入 SYN_RCVD 状态，等待握手的第三次 ACK。由于源地址伪造，服务器永远等不到 ACK，其半连接队列（backlog 队列）被快速填满，导致合法用户的 SYN 请求被丢弃，正常服务不可用。其本质是滥用 TCP 协议设计中的“无状态请求引发有状态资源分配”这一不对称性：攻击者只需发送几十字节的无状态报文，即可迫使服务器分配大量内存并维持定时器。防御的核心思路是降低资源分配成本、缩短超时时间、或利用 SYN Cookie 将连接状态编码在序列号中，使服务器在收到 ACK 前不分配任何资源。该知识点位于网络协议栈与系统安全的交汇处，是理解 DoS/DDoS 攻击原理、操作系统内核网络栈、以及负载均衡/防火墙/云防护等基础设施设计的基础。专业工程师必须掌握它，因为任何面向公网的 TCP 服务都可能暴露于此风险，且它深刻体现了协议设计与资源管理之间的权衡，是衡量系统设计者底层功底的关键场景。

### 2. 底层原理剖析
标准 TCP 三次握手：客户端发送 SYN → 服务器回复 SYN-ACK 并进入 SYN_RCVD，同时将连接请求放入半连接队列（如 Linux 中的 syn queue）→ 客户端回复 ACK → 连接迁移到全连接队列（accept queue）→ 应用 accept()。攻击时，攻击者发送源地址随机伪造的 SYN，服务器收到后回复 SYN-ACK 到伪造地址，然后等待 ACK。由于源地址不存在，服务器在超时（通常数秒到数十秒）前，该半连接一直占用 TCB 和队列槽位。当半连接队列满后，新到达的 SYN（包括合法用户的）被内核直接丢弃。本质问题是：服务器为每一个不可信的 SYN 报文立刻分配了完整的内存资源和定时器，而攻击者几乎零成本。防御机制分层：1. 增大队列并缩短 SYN_RCVD 超时（如 Linux 的 tcp_synack_retries 降低），只能延缓，不能根治。2. SYN Cookie：不分配 TCB，而是将连接参数（源/目的 IP、端口、一个服务器密钥）通过哈希生成一个 32 位序列号作为 SYN-ACK 的初始序列号。服务器不保存任何状态。当客户端回 ACK 时，其确认号必须等于该序列号+1，服务器通过逆运算校验，若合法则直接分配 TCB 并建立连接。攻击者无法伪造源 IP 对应的 ACK（除非截获或猜测），因此合法连接不受影响。SYN Cookie 的代价是丢失了部分 TCP 选项（如大窗口、SACK），因此 Linux 在队列满时才启用。3. 其他：反向探测（向 SYN 源发探测包，不响应则丢弃）、SYN Proxy（代理代答握手，验证通过后再与后端建连）、限速（按源 IP 限 SYN 速率）。与前端已有概念的对比：类似于 TypeScript 的接口与 Java 的接口的区别——TS 接口是编译期的结构性类型契约，编译后完全消失，无运行时开销；Java 接口是运行时多态的基础，存在类加载和方法分派成本。SYN Cookie 的设计本质是“把状态从服务器内存中抹去，编码到协议本身的字段里”，与 HTTP 无状态会话用 JWT 编码用户状态有异曲同工之妙，但 TCP 的字段更底层、更受限。另一个类比：前端中 React 的协调（Reconciliation）为避免为每个 DOM 变化分配新组件实例而采用 diff 复用，而 SYN Flood 防御则是为了避免为每个连接请求分配内核对象而采用计算代替存储。

### 3. 基础代码与实战验证
以下为 Linux 内核 SYN Cookie 核心逻辑的极简伪代码（基于真实实现抽象）：

```
// 服务器收到 SYN 时，若 syn queue 已满，则启用 SYN Cookie
void syn_rcv(struct sk_buff *skb) {
    struct tcphdr *th = tcp_hdr(skb);
    // 1. 计算 cookie：本质是一个 hash，包含双方 IP/端口和一个密钥
    //    syncookie_secret 是系统启动时生成的随机密钥，防止伪造
    __u32 cookie = cookie_v4_init_sequence(skb, th);
    // 2. 直接构造 SYN-ACK 并发送，不分配 TCB、不插入任何队列
    //    注意：SYN-ACK 的序列号直接使用 cookie
    tcp_send_synack(sk, skb, cookie);
    // 函数返回，无任何状态残留
}

// 服务器收到 ACK 时，从确认号中提取 cookie 并校验
int tcp_v4_rcv(struct sk_buff *skb) {
    // 3. 从 ACK 的确认号减 1 得到 cookie（因为 ACK 号 = 序列号 + 1）
    __u32 cookie = ntohl(th->ack_seq) - 1;
    // 4. 重新计算 cookie，并与客户端返回的比对
    __u32 computed = cookie_v4_init_sequence(skb, th);
    if (cookie == computed) {
        // 5. 校验通过，此时才真正分配 TCB，建立连接
        tcp_create_openreq_child(sk, skb, req);
    } else {
        // 校验失败，直接丢弃，不占任何资源
        kfree_skb(skb);
    }
}
```

关键行注释：
- `__u32 cookie = cookie_v4_init_sequence(skb, th);` 这一行是整个防御的核心——将连接状态（四元组）压缩成一个 32 位整数，服务端不保存任何半连接信息，攻击者发送的海量 SYN 在这里被转化为无状态的“计算”，而计算成本远低于内存分配。
- `tcp_send_synack(sk, skb, cookie);` 不分配 TCB，直接发送 SYN-ACK，此时攻击者伪造源地址也无妨，因为不会有资源被占用。
- 校验时的重新计算是“无状态验证”的精髓：服务器无需查询任何表，只需用同样的算法和密钥重新计算一次，即可确认该 ACK 是否对应一个真实的握手流程。

实际验证方式：在 Linux 上通过 `sysctl -w net.ipv4.tcp_syncookies=1` 开启，并用 `hping3 -S -p 80 --flood -i u1000 <target>` 攻击，观察服务器 `ss -s` 中 SYN_RECV 数量不会暴涨，而 `netstat -s` 中 `SYNs to LISTEN sockets dropped` 计数增加。
