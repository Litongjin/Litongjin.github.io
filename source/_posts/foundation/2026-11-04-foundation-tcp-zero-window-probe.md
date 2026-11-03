---
title: "每日基础技术总结 · 2026-11-04 · TCP零窗口探测（Zero Window Probe）的定时器触发与避免死锁机制"
date: 2026-11-04 07:01:40
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-04 · TCP零窗口探测（Zero Window Probe）的定时器触发与避免死锁机制

## 📚 今日主题

> **TCP零窗口探测（Zero Window Probe）的定时器触发与避免死锁机制**（网络基础）

### 1. 核心概念速览
### 1. 核心概念速览
TCP 零窗口探测（Zero Window Probe，ZWP）是 TCP 流量控制子系统的可靠性补充机制。接收方通过 TCP 首部 16 位窗口字段（经窗口缩放选项可扩展）通告当前接收缓冲可用空间 RCV.WND；当 RCV.WND=0 时，发送方必须停止发送有效数据。若接收方在应用读取数据后发出的窗口更新 ACK 丢失，发送方会持续认为窗口为 0，接收方则等待新数据，形成传输层死锁。发送方在 SND.WND=0 且仍有待发数据时启动 persist timer（坚持定时器），超时后发送零窗口探测报文，强制接收方返回携带当前窗口的 ACK；若窗口仍为 0，则按指数退避重试，直到窗口打开或连接因重试超限终止。其本质是用周期性探测同步控制平面状态，解决窗口更新 ACK 不可靠导致的数据平面停滞。
它位于 TCP 定时器与流控机制中，与 RTO 重传、delayed ACK、keepalive、SWS 避免、拥塞控制相邻但职责不同。后端高并发服务、代理网关、消息队列、数据库连接池以及 AI 推理流式响应中的慢消费者场景，都可能出现零窗口；掌握该机制才能正确解释 send 阻塞/EAGAIN、吞吐骤降、连接假死以及 tcpdump 中的 ZeroWindow/ZeroWindowProbe。

### 2. 底层原理剖析
### 2. 底层原理剖析
发送方维护 SND.UNA、SND.NXT、SND.WND。可发送上界为 SND.UNA + SND.WND；当 SND.NXT 到达该上界且 SND.WND=0 时，发送方停止发送。触发条件不是「看到零窗口」本身，而是「SND.WND=0 且存在待发送数据」。此时启动 persist timer，而不是 RTO 重传定时器。
定时器到期后的状态迁移：
1. 发送零窗口探测。通常发送 1 字节新数据（序号 SND.NXT），部分实现可发送已确认旧数据；探测是窗口为 0 时允许发送的例外报文。
2. 接收方收到探测后必须回 ACK，即使 RCV.WND 仍为 0；ACK 携带当前 RCV.WND。若探测字节因窗口为 0 被丢弃，RCV.NXT 不推进；若窗口已打开且探测字节被接受，RCV.NXT 推进。
3. 发送方收到 ACK：若窗口仍为 0，重置 persist timer，间隔按 RTO 指数退避并受内核上限约束（Linux 约 120s，具体受版本与 sysctl 影响）；若窗口大于 0，取消 persist timer，更新 SND.WND，恢复发送。
4. 探测不增加拥塞窗口，不触发拥塞控制，也不等同于数据重传；其目的是获取窗口状态。超过内核重试上限后连接被判定异常。
避免死锁的关键在于：接收方窗口打开时主动发送窗口更新 ACK 可能丢失或被延迟，persist 探测强制接收方重新表达当前窗口，使发送方与接收方的 SND.WND/RCV.WND 最终一致。
与前端已有概念对比：浏览器 Streams 的 desiredSize/pull 是应用层显式背压，由 JS 代码控制读取节奏；TCP 窗口是内核隐式背压，应用无感知。前端 fetch 轮询重试由应用定时器触发，无传输层 ACK 与序号语义；persist timer 由内核基于 SND.WND 和 ACK 状态触发，探测报文不交付应用。WebSocket 心跳是应用层 PING/PONG，用于空闲连接存活检测；零窗口探测用于有数据但窗口为 0 的流控死锁恢复。类似 Java interface 与 TS interface 的差异：应用层重试只是「声明式策略」，内核 persist 是运行时真实状态机与报文确认。

### 3. 基础代码与实战验证
```text
### 3. 基础代码与实战验证
以下用 Python 标准库复现：接收端缩小 SO_RCVBUF 且不 recv，使内核通告窗口降为 0；发送端填满窗口后保持连接，由内核自动触发 persist timer。
receiver.py:
import socket, time
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
s.setsockopt(socket.SOL_SOCKET, socket.SO_RCVBUF, 2048)  # 缩小接收缓冲，使 RCV.WND 更快降为 0
s.bind(('127.0.0.1', 9000))
s.listen(1)
conn, addr = s.accept()
time.sleep(120)  # 不调用 recv：接收队列填满后内核通告窗口 0；收到探测只回 ACK，不交付应用

sender.py:
import socket, time
c = socket.create_connection(('127.0.0.1', 9000))
c.setblocking(False)
chunk = b'x' * 65536
sent = 0
end = time.time() + 5
while time.time() < end:
    try:
        sent += c.send(chunk)  # 受 SND.WND 限制；窗口耗尽后返回 EAGAIN
    except BlockingIOError:
        time.sleep(0.1)  # 应用层退避；内核 persist timer 独立运行
print(sent)
time.sleep(120)  # 保持连接，等待内核 ZeroWindowProbe 指数退避

验证命令：
python3 receiver.py &
python3 sender.py &
sudo tcpdump -i lo -nn -S 'tcp port 9000'
ss -tio state established '( sport = :9000 or dport = :9000 )'
关键观察：tcpdump 中可见 win 0、ZeroWindowProbe、ZeroWindowProbeAck（不同工具命名略有差异）；ss 中可见 timer:(persist,...)。探测由内核 TCP 栈自动完成，应用层只看到 send 阻塞或 EAGAIN。
```

### 4. 常见误区与进阶思考
### 4. 常见误区与进阶思考
误区一：把零窗口探测等同于 keepalive 或普通重传。keepalive 解决空闲连接对端存活检测；零窗口探测解决窗口更新丢失导致的流控死锁。探测不增加拥塞窗口，不触发拥塞控制，也不保证探测字节被应用接收。
误区二：认为接收方应用一旦读取数据就会立即发送窗口更新，因此不会死锁。窗口更新 ACK 可能丢失或被延迟；发送方在收到非零窗口前仍认为 SND.WND=0，必须依靠 persist 探测重新同步。另一个误区是认为探测间隔固定，实际按 RTO 指数退避并受内核上限与重试次数约束。
思考题：接收方窗口从 0 变为正后主动窗口更新 ACK 丢失；发送方 persist 探测到达时接收方窗口已打开。若探测发送的是 1 字节新数据（序号 SND.NXT），接收方会接受并推进 RCV.NXT，发送方如何根据 ACK 号与窗口字段恢复发送且不产生空洞？若探测改为发送已确认旧数据（序号 SND.UNA-1），接收方回 ACK 时 ACK 号与窗口字段又如何影响发送方对 SND.UNA/SND.NXT 的判断？请比较两种探测策略在「窗口仍为 0」与「窗口刚打开」两种状态下的副作用。
