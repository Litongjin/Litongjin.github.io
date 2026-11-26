---
title: "每日基础技术总结 · 2026-11-24 · TCP 滑动窗口协议中的零窗口探测 (Zero Window Probe) 机制与超时重传"
date: 2026-11-24 08:00:00
categories: [技术分享]
tags: ["技术分享", "操作系统基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-24 · TCP 滑动窗口协议中的零窗口探测 (Zero Window Probe) 机制与超时重传

## 📚 今日主题

> **TCP 滑动窗口协议中的零窗口探测 (Zero Window Probe) 机制与超时重传**（操作系统基础）

### 1. 核心概念速览
Zero Window Probe 是 TCP 发送端在对端通告 rwnd=0 后，为防止『窗口更新 ACK 丢失导致发送端永久停发』而进入 persist 状态，并按指数退避周期性发送极小探测段，强制接收端返回当前 rwnd 的机制。它的本质不是恢复丢失的数据，而是恢复丢失的流量控制状态。超时重传则是发送端基于 RTO 对已发送数据段未获 ACK 的可靠性恢复。二者同属传输层可靠传输状态机：RTO 处理数据/ACK 丢失，Zero Window Probe 处理窗口状态丢失。它位于操作系统内核网络栈，是 HTTP 长连接、RPC 流、对象存储上传、AI 训练参数同步与推理流式输出中背压、吞吐退化、连接假死问题的底层根因之一。专业工程师必须掌握，否则无法区分网络丢包、接收消费慢、内核缓冲耗尽与协议状态机死锁。

### 2. 底层原理剖析
发送端维护 SND.UNA、SND.NXT、SND.WND、cwnd。正常发送上限是 min(SND.WND, cwnd)。收到 ACK 后更新 SND.WND；若 window=0，进入 zero-window persist：停止发送新数据，启动 persist timer。timer 到期且窗口仍为 0 时，发送一个 1 字节探测段，序号使用最早未确认字节（SND.UNA 或等价边界）；接收端即使缓存为 0，也必须返回 ACK，并在 ACK 中给出当前真实窗口。若仍为 0，persist 间隔指数退避；若窗口非 0，退出 persist 并恢复发送。
普通超时重传是另一条路径：发送数据后维护 RTO；RTO 由 SRTT 与 RTTVAR 估算（RTO = SRTT + 4 * RTTVAR），重传段不用于 RTT 采样（Karn），超时后重传 SND.UNA 对应完整段，执行拥塞控制损失恢复并指数退避。Zero Window Probe 与 RTO 都使用定时器，但故障对象不同：RTO 恢复丢失的数据段/ACK，Zero Window Probe 恢复丢失的窗口状态；前者通常触发拥塞窗口收缩，后者不是拥塞事件。
接收端处理：收到探测时若缓冲区仍满，可以丢弃探测字节，但必须回 ACK；若应用已消费出空间，则 ACK 携带非零 rwnd，发送端立即解除暂停。前端对比：ReadableStream/WritableStream 的 highWaterMark 是应用层背压，TCP rwnd 是传输层背压；fetch timeout/AbortSignal 是应用层整体超时，TCP 的 RTO 与 persist timer 是协议状态机对不同故障的细分。与 TS interface/Java interface 的差异类似：TS interface 是编译期结构约束，运行时无状态；TCP 窗口是运行时动态契约，Zero Window Probe 是运行时主动探测并刷新契约的机制。

### 3. 基础代码与实战验证
```text
// verify.js：制造接收端不消费数据，使内核把 rwnd 压到 0，从而观察 persist/zero-window 探测
const net = require('net');
const PORT = 9000;

if (process.argv[2] === 'server') {
  const server = net.createServer(conn => {
    try { conn.setRecvBufferSize(4096); } catch (_) {}
    conn.pause(); // 关键：应用层不 read()，内核接收队列无法排空，接收窗口最终降为 0
    console.log('server: accepted and paused');
  });
  server.listen(PORT, () => console.log('server listening'));
} else {
  const client = net.connect(PORT, '127.0.0.1', () => {
    console.log('client connected');
    const chunk = Buffer.alloc(64 * 1024, 0x41);
    let total = 0;
    const timer = setInterval(() => {
      total += chunk.length;
      const ok = client.write(chunk); // write 只进入本地发送队列；实际发送受对端 rwnd 与 cwnd 限制
      if (!ok) console.log('client: local send buffer backpressure', total);
    }, 10);
    client.on('error', err => console.error(err.message));
  });
}

# Linux 验证：
# 观察发送端 TCP 定时器；出现 persist 表示进入零窗口探测状态。
watch -n 0.5 'ss -tin | grep :9000'
# 抓包查看 win 0 的 ACK，以及后续极小载荷的探测/确认交互。
sudo tcpdump -i lo -nn -S -v 'tcp port 9000'
```

### 4. 常见误区与进阶思考
误区一：把 persist/zero-window 计数直接归因为网络丢包或链路差。它更常指向接收端应用消费慢、用户态未读取、内核接收缓冲耗尽；应先分析应用读取路径、事件循环阻塞、GC、read backlog、SO_RCVBUF，而不是只调大发送端超时。
误区二：认为 rwnd=0 后发送端只需等待窗口更新。窗口更新是普通 ACK，可能丢失；若无 Zero Window Probe，发送端永久等待会形成死锁。
进阶思考：为什么典型实现让零窗口探测携带 1 字节数据，而不是纯 0 字节窗口查询？请结合序号语义、重复字节必须触发 ACK、旧探测与窗口更新竞争、以及中间设备对零载荷控制段的处理差异，说明 1 字节如何提高状态恢复的确定性。
