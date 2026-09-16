---
title: "每日基础技术总结 · 2026-09-17 · TCP 三次握手 / 四次挥手"
date: 2026-09-17 07:01:29
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-17 · TCP 三次握手 / 四次挥手

## 📚 今日主题

> **TCP 三次握手 / 四次挥手**（前端底层与计算机基础）

### 1. 核心概念速览
定义：TCP 三次握手（Three-Way Handshake）是 TCP 在传输数据前，双方内核通过交换三个带控制标志位的报文段（SYN → SYN+ACK → ACK），完成初始序列号（ISN）双向同步与可选参数协商，并在各自协议栈内建立一对相互一致的 TCB（Transmission Control Block），使双方进入 ESTABLISHED 的过程。四次挥手（Four-Way Termination）是全双工连接的两个数据传输方向各自独立发送 FIN 并各自被确认（FIN → ACK → FIN → ACK），最终双方释放 TCB；主动关闭方在发出最后一个 ACK 后进入 TIME_WAIT，等待 2MSL 后回到 CLOSED。

本质：所谓「连接」不是物理链路，而是内核中的一个状态对象，加上双方对「字节流编号起点、已确认字节、窗口/拥塞/选项参数」的一致认知。三次握手的产物不是「通路」，而是两个内核中一致的四元组 TCB；它解决的是：在 IP 这种不可靠、无连接、可能重复与乱序的承载之上，为两条方向相反的字节流建立可靠的编号基准与状态机，同时抵御网络中滞留的历史报文。

挥手解决的是：全双工意味着两个方向的数据流必须能独立终止（半关闭），必须保证「最后一段数据 + 最后一个 FIN」被可靠确认，同时必须为被释放的四元组留出报文消散窗口，避免迟到的旧报文被复用同一四元组的新连接误收。

体系位置：TCP 位于 IP 之上，用序号/确认/重传/滑动窗口/拥塞控制在不可靠 IP 之上构造「可靠、有序、全双工字节流」抽象；向上支撑 HTTP/1.1、HTTP/2、TLS、WebSocket、gRPC、MySQL、Redis 等几乎所有主流应用协议（QUIC 是例外，基于 UDP 自建握手与连接标识）。AI 侧的 gRPC 推理服务、参数服务器与分布式训练的 TCP 后端、向量库客户端同样建筑于此。

为什么必须掌握：连接建立与释放直接决定 RTT 数量（建连固定 1.5 RTT 开销）、半连接/全连接队列容量、TIME_WAIT 端口占用与慢启动成本，这些直接映射到前端 TTFB、首屏与长连接治理；线上多数「连接超时 / connection reset / CLOSE_WAIT 堆积 / 端口耗尽 / 偶发串包」问题，不懂状态机就只能靠重试掩盖。

### 2. 底层原理剖析
一、报文语义与序号规则
- seq 为 32 位，对字节流编号；ISN 按 RFC 6528 基于时钟 + 四元组哈希随机生成，防止被猜测与旧连接串包。
- SYN 与 FIN 各消耗 1 个序列号（即便不携带数据）；不带数据的纯 ACK 不消耗。
- ACK 号是累积确认：ACK = x 表示「x 之前的字节全部收到，期望下一个字节序号为 x」；重复 ACK 是快速重传与拥塞控制的输入。

二、状态迁移与内核伪代码
客户端内核：
CLOSED → 建 TCB、选 ISN=x → send SYN(seq=x, MSS/WS/SACK/TS) → SYN_SENT
收到 SYN+ACK，校验 ack == x+1 且 SYN 置位 → 发 ACK(ack=y+1) → ESTABLISHED（connect 返回 / epoll 可写）
若收到 RST → ECONNREFUSED

服务端内核：
LISTEN → 收 SYN(x) → 建半连接 TCB，选 ISN=y，state=SYN_RCVD，回 SYN(seq=y, ack=x+1) → 入 SYN 队列（受 tcp_max_syn_backlog 限制，溢出则丢包或启用 syncookies）
收 ACK(ack=y+1) → 移入 accept 队列（受 listen backlog / somaxconn 限制）→ ESTABLISHED；accept() 只是从该队列取走已就绪 socket
SYN+ACK 未确认 → 指数退避重传（tcp_synack_retries，默认约 1+2+4+8+16 = 31s）→ 删除 TCB

三、为什么恰好是三次
- 两次不够：服务端发出 SYN+ACK 后，只能确认「自己能发」与「对方能发」，无法确认「对方已收到我的 ISN」。若该 SYN 是网络中滞留的旧连接重复报文，服务端会凭空建立无人使用的连接并占用资源，而客户端会回 RST。
- 四次多余：服务端的 SYN 与对客户端的 ACK 无先后依赖，可合并为一个报文（SYN+ACK），故最小代价为 1.5 RTT。
- 第三次 ACK 的唯一职责：向服务端提供「客户端已接受我的 ISN、双向 ISN 同步完成」的证据，并确认本轮握手不是历史连接。

四、四次挥手
FIN 的语义是「我不会再发数据了」，不是「我不能收数据」，因此是半关闭。
主动方：ESTABLISHED →（应用层 close）发 FIN → FIN_WAIT_1 → 收 ACK → FIN_WAIT_2 → 收对端 FIN → 回 ACK → TIME_WAIT(2MSL) → CLOSED
被动方：ESTABLISHED → 收 FIN → 内核立即回 ACK → CLOSE_WAIT（应用层读到 EOF/read 返回 0）→（应用层 close）发 FIN → LAST_ACK → 收 ACK → CLOSED
- 为什么通常四次：被动方内核收到 FIN 立刻回 ACK，但本端 FIN 必须等应用层处理完残留数据、调用 close 之后才发，中间隔着不可预测的应用层时间，无法与 ACK 合并。
- 何时退化为三次：应用层收到 EOF 后立即 close（或 shutdown(SHUT_WR)），且对端延迟确认刚好覆盖，FIN 可捎带 ACK。
- TIME_WAIT 为何是 2MSL（Linux 固定 60s，MSL=30s）：①若最后 ACK 丢失，对端会重传 FIN，本端仍能重发 ACK（1 个 MSL 覆盖 ACK 到达，1 个 MSL 覆盖重传 FIN 返回）；②让本连接的所有报文在网络中消亡，防止新连接复用同一四元组时被旧数据污染。收到重复 FIN 会重发 ACK 并重置计时器。

五、异常路径
- 目标端口无监听 → RST，connect 立即 ECONNREFUSED。
- 向已关闭连接发数据 → RST，本端读出 ECONNRESET。
- SO_LINGER(l_onoff=1,l_linger=0) 或 Node 的 socket.destroy() → 直接 RST，跳过四次挥手（无 TIME_WAIT，但对端可能丢数据）。
- 半开连接（对端进程崩溃/断线）只能靠内核 KeepAlive（tcp_keepalive_time 默认 2h）或应用层心跳发现。

六、与前端已有概念的对照
- 与 TS / Java interface 对照：TS interface 是编译期结构化类型约束，运行时被 erase，零运行时成本；TCP 握手是运行时双向状态协商，必须付出 1.5 RTT 的真实成本，结果保存在内核而非编译器符号表。前者是「编译期静态契约」，后者是「运行时动态状态同步」——同一个「接口/握手」词，一个在编译期，一个在网络协议栈。
- 与 WebSocket 握手对照：WS 的 Upgrade 是应用层语义（HTTP/1.1 101 Switching Protocols），建立在已完成三次握手的 TCP 之上，二者不同层次；ws 库中的 handshake 代码不会产生任何 SYN。
- 与浏览器 performance 对照：Resource Timing 中 connectStart/connectEnd 区间大致对应 TCP 建连（TLS 由 secureConnectionStart 划分），domainLookup 对应 DNS；用 fetch 也只能观测握手耗时，无法在 JS 层介入。
- 与 EventLoop 对照：connect() 由内核完成握手后经 epoll 通知 libuv 才回调，因此 Node 的 'connect' 回调触发时 ESTABLISHED 已成事实；这也解释了为什么 JS 无法「取消」握手，只能 destroy → RST。

### 3. 基础代码与实战验证
```text
// server.js —— node server.js 先启动
const net = require('net');
const server = net.createServer({ allowHalfOpen: true }, (socket) => {
  // 回调被调用 = 三次握手已完成：内核 TCB 已从 SYN_RCVD 迁入 ESTABLISHED 并移出 accept 队列
  console.log('[srv] ESTABLISHED remote=' + socket.remoteAddress + ':' + socket.remotePort);
  socket.on('data', (b) => console.log('[srv] recv data: ' + b.toString().trim()));
  socket.on('end', () => {
    // 收到对端 FIN：内核已自动回 ACK，本端进入 CLOSE_WAIT；此时本端仍可 write（半关闭）
    console.log('[srv] got FIN -> CLOSE_WAIT');
    socket.end(); // 本端发送 FIN -> LAST_ACK；收到对端 ACK 后 -> CLOSED，TCB 释放
  });
  socket.on('close', () => console.log('[srv] CLOSED'));
});
server.listen(9000, '127.0.0.1');

// client.js —— node client.js
const net = require('net');
const s = net.connect({ host: '127.0.0.1', port: 9000 }, () => {
  // 回调触发时内核已完成第三次握手：SYN -> SYN+ACK -> ACK，本地端口已绑定并进入 ESTABLISHED
  console.log('[cli] ESTABLISHED localPort=' + s.localPort);
  s.write('hello\n');
});
s.setNoDelay(true); // 关闭 Nagle，避免小包被合并，便于抓包逐包观察
s.on('data', (b) => console.log('[cli] recv: ' + b.toString().trim()));
s.on('end', () => console.log('[cli] got FIN -> 半关闭'));
s.on('close', () => console.log('[cli] closed'));
setTimeout(() => s.end(), 500); // end() 发 FIN：本端成为主动关闭方 -> FIN_WAIT_1 -> FIN_WAIT_2 -> TIME_WAIT

# 观察状态机（Linux）
ss -tanp | grep 9000          # ESTABLISHED / FIN-WAIT-2 / TIME-WAIT / CLOSE-WAIT 一目了然
ss -tan state time-wait       # 客户端 localPort 会在 TIME-WAIT 停留约 60s（内核 TCP_TIMEWAIT_LEN = 2MSL）
# macOS: netstat -anv -p tcp | grep 9000

# 抓包看报文（-S 显示绝对序号，可验证 SYN 与 FIN 各占 1 个序列号）
sudo tcpdump -i lo -nn -S 'tcp port 9000'
# Flags [S]  seq=x             <- 第 1 次握手
# Flags [S.] seq=y ack=x+1    <- 第 2 次握手（SYN 与 ACK 合并，故只需 3 个报文）
# Flags [.]  ack=y+1          <- 第 3 次握手，双向 ISN 同步完成
# Flags [F.] ...              <- 主动方 FIN（常捎带对端数据的 ACK）
# Flags [.]  ack=...          <- 被动方确认
# Flags [F.] ...              <- 被动方 FIN
# Flags [.]  ack=...          <- 主动方最终 ACK，随后进入 TIME_WAIT

# 对照实验一：把 client 的 s.end() 换成 s.destroy()，抓包可见 Flags [R]，无四次挥手、无 TIME_WAIT
# 对照实验二：去掉 server 的 allowHalfOpen，Node 默认收到 FIN 立即回 FIN，挥手在抓包上退化为三次
# 对照实验三：把 server 的 listen backlog 与 net.ipv4.tcp_max_syn_backlog 调小并施压，可复现 SYN 被丢弃/超时
```

### 4. 常见误区与进阶思考
误区 1（最高频）：把「三次握手是为了确认双方收发能力」当成本质。这只是副产品而非目的，且解释力不足。握手的核心产物是双方内核中一致的 TCB 与双向 ISN 同步，第一职责是防止网络中滞留的历史重复 SYN 建立一个错误连接。反问检验：若只为确认收发能力，为什么第二次握手后服务端不能直接 ESTABLISHED 并开始发数据？因为此时服务端尚不能确认自己的 ISN 被对端接受；若对端根本不认这个连接（历史 SYN），服务端会持有一个幽灵 TCB，甚至把旧连接的滞留数据当作本连接数据。另外，MSS、窗口缩放因子、SACK permitted、时间戳等选项只在 SYN 中协商，握手同时是单向参数声明过程，不是「测试链路」。

误区 2：把 TIME_WAIT 当作内核缺陷，用错误参数去「优化」。①tcp_fin_timeout 控制的是 FIN_WAIT_2，不是 TIME_WAIT；TIME_WAIT 时长由内核常量 TCP_TIMEWAIT_LEN 固定为 60s，无法通过 sysctl 修改；②tcp_tw_recycle 依赖 per-host 时间戳，在 NAT 场景会误丢握手包，已于 Linux 4.12 移除；tcp_tw_reuse 只对本机主动外连生效且依赖时间戳单调递增；③TIME_WAIT 是正确性机制而非泄漏，真正该做的是减少主动关闭次数（HTTP keep-alive、连接池、长连接心跳）；④反向的 CLOSE_WAIT 堆积从来不是内核问题，而是应用层收到 EOF 后没有 close/destroy（例如 Node 中监听了 end 却没调用 end()/destroy()，或 keep-alive 连接未被回收），进程会持续持有 fd 直至耗尽。

误区 3（次要）：认为四次挥手必然四次。延迟确认叠加应用层立即关闭会合并为三次；反之若对端在挥手前仍有数据要发，挥手会被拉长，甚至出现 FIN_WAIT_2 长时间驻留（对端迟迟不发 FIN，需靠 tcp_fin_timeout 兜底）。

思考题：为什么 TIME_WAIT 必须由主动关闭方承担？假设协议规定改由被动关闭方持有 2MSL，请在以下三条路径上逐步推演报文时序并说明失效点：①最后一个 ACK 丢失、对端重传 FIN 时，谁能重发 ACK；②本机以相同四元组快速发起新连接时，旧报文由谁负责吸收；③对端在旧连接上滞留的报文晚到时，会被哪个连接的接收窗口误收。
