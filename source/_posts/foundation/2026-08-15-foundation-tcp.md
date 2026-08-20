---
title: "每日基础技术总结 · 2026-08-15 · TCP 的慢启动与拥塞避免"
date: 2026-08-15 08:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-08-15 · TCP 的慢启动与拥塞避免

## 📚 今日主题

> **TCP 的慢启动与拥塞避免**（网络基础）

### 1. 核心概念速览
TCP 拥塞控制的核心目标是防止网络因注入过多数据而瘫痪，慢启动（Slow Start）与拥塞避免（Congestion Avoidance）是其中两个相邻的、由 cwnd（拥塞窗口）驱动的状态阶段。慢启动的本质是：在连接初始阶段，由于对可用带宽一无所知，TCP 采用指数增长探测带宽上限；每收到一个 ACK，cwnd 增加 1 个 SMSS（发送方最大报文段大小），因此每 RTT 内 cwnd 翻倍。当 cwnd 达到 ssthresh（慢启动阈值）时进入拥塞避免阶段，此阶段 cwnd 每 RTT 仅增加 1 个 SMSS，呈线性增长，用于在接近可用带宽上限时缓慢逼近，避免再次触发丢包。这套机制解决的是分布式网络环境下，发送端如何在没有全局链路状态信息的情况下，自适应地确定一个既不过载又高效的发送速率。它在整个 TCP/IP 协议栈中位于传输层，是 TCP 可靠传输与流量控制之外的第三大支柱——网络拥塞控制，直接影响互联网的公平性与稳定性。专业工程师必须掌握它，因为它是理解 CDN 性能优化、长连接池调优、分布式系统网络瓶颈以及云原生服务限流设计的底层依据；前端工程师接触的 HTTP/2、HTTP/3 的拥塞控制行为（如 BBR、CUBIC）均与这些基础状态机一脉相承，忽略它们就无法从原理层面解释为何某些网络请求慢、为何修改 TCP 缓冲区能提升吞吐。

### 2. 底层原理剖析
TCP 拥塞控制的状态机由 ssthresh 与 cwnd 共同驱动。初始时 ssthresh 为任意大值（如系统默认或对端通告窗口），cwnd 从初始窗口（通常 10 个 MSS 或按 RFC 5681 建议）开始。核心运行逻辑如下伪代码：

```
每次收到 ACK:
    if (cwnd < ssthresh):
        # 慢启动：每 ACK 增加 1 MSS，指数增长
        cwnd += MSS
    else:
        # 拥塞避免：每 RTT 增加 1 MSS，线性增长
        cwnd += MSS * MSS / cwnd
```

注意：这里'每 ACK 增加 1 MSS'的累积效果是，在一个 RTT 内若有 cwnd/MSS 个 ACK 返回，cwnd 增加 cwnd/MSS * MSS = cwnd，即翻倍。而拥塞避免的增量公式在 cwnd 较大时每次 ACK 的增量很小，累加一个 RTT 恰好是 1 MSS。

丢包检测触发重传时，TCP 进入拥塞控制的状态调整：若收到 3 个重复 ACK（快速重传），则 ssthresh = max(cwnd/2, 2*MSS)，cwnd = ssthresh（快速恢复阶段，配合拥塞避免线性增长）；若超时重传，则 ssthresh = max(cwnd/2, 2*MSS)，cwnd = MSS（重新从慢启动开始）。这套机制的底层逻辑是：丢包被作为拥塞发生的唯一可靠信号（早期设计），通过乘法减半和加法增长，实现 AIMD（加法增大，乘法减小），从而保证多个 TCP 流共享瓶颈链路时的收敛性与公平性。

与前端已有的概念对比：TCP 的慢启动类似浏览器中的连接预热——初始并发请求数有限，待首个响应确认链路可用后再逐步提高并发（HTTP/1.1 的 6 连接限制本质是对建立新 TCP 慢启动成本的人为规避）；而拥塞避免更接近 React 的批处理——当状态更新频繁时，合并更新以减少开销，但 TCP 中它是按 RTT 粒度线性调度，本质是'试探-反馈-调整'闭环，不同于前端常用的配置化或预设阈值控制。另一个对比：Java 的接口与 TypeScript 的接口虽同名但语义不同，前者是运行时多态契约，后者是编译期结构约束；类似地，慢启动与流控制（流量控制）虽都控制发送速率，但流控制基于接收方缓冲区剩余空间，是端到端反馈；而拥塞控制基于网络中间节点承受能力，是路径状态推断。二者通过 cwnd 与 rwnd 取最小值共同决定实际发送窗口，缺一不可。

### 3. 基础代码与实战验证
由于 TCP 拥塞控制运行在内核协议栈，无法在应用层直接用纯代码调用，但可以用一个离散事件模拟器精确复现其状态机逻辑。以下为极简 Python 模拟，展示 cwnd 在慢启动与拥塞避免阶段的演变。

```python
import random

def tcp_cwnd_simulation():
    # 初始化：cwnd 初始 10 个 MSS，ssthresh 设为 64 MSS
    cwnd = 10
    ssthresh = 64
    rtt = 100  # 模拟每个 RTT 固定为 100ms
    time = 0
    for rtt_num in range(30):
        # 模拟一个 RTT 内发送 cwnd 个 MSS，全部 ACK 返回（无丢包）
        acked = cwnd
        if cwnd < ssthresh:
            # 慢启动：每个 ACK 增加 1 MSS，本 RTT 共增加 acked 个 MSS
            cwnd += acked
        else:
            # 拥塞避免：本 RTT 共增加 1 MSS，用增量公式保证每 RTT 恰好 +1
            cwnd += 1  # 等价于 MSS*MSS/cwnd 的累计效果（cwnd 以 MSS 为单位）
        time += rtt
        print(f"time={time}ms cwnd={cwnd} MSS, ssthresh={ssthresh}")

# 输出节选：cwnd 将从 10 按指数增长至超过 ssthresh 后转为线性增长
tcp_cwnd_simulation()
```

关键行注释：
- `cwnd += acked`：慢启动中，一个 RTT 内收到 cwnd 个 ACK，每个 ACK 触发 cwnd+1，因此 cwnd 翻倍，这是指数增长的本质。
- `cwnd += 1`：拥塞避免中，一个 RTT 内累计增加 1 MSS，保证加法增长，避免快速逼近网络极限。

若要验证丢包后的行为，可加入 3 个重复 ACK 触发快速重传：

```python
def on_dup_ack():
    global cwnd, ssthresh
    ssthresh = max(cwnd // 2, 2)
    cwnd = ssthresh  # 快速恢复阶段
```

该代码展示了 AIMD 的核心状态迁移。实际工程中，通过 `ss`、`tcpdump` 或内核 eBPF 可以观测真实 TCP 流的 cwnd 变化，但上述模拟足以理解机制本身。

### 4. 常见误区与进阶思考
误区一：认为慢启动只发生在连接建立时。实际上，每当 TCP 连接发生超时重传，或空闲一段时间后（如 HTTP/1.1 连接复用后的重启），TCP 会重置 cwnd 并重新进入慢启动，即使该连接已经存在很久。前端工程师在优化长连接时，如果忽略了空闲连接重入慢启动的成本，就会误判复用连接一定比新建连接快；因此 Keep-Alive 的保活机制需要配合 TCP 的 `TCP_USER_TIMEOUT` 或应用层心跳来维持窗口不衰减。

误区二：混淆拥塞窗口与接收窗口。cwnd 是发送方根据网络拥塞程度推算的窗口，rwnd 是接收方通告的可用缓冲区大小，实际发送窗口 = min(cwnd, rwnd)。若只关注内核 socket buffer 大小（rwnd）而忽略 cwnd，即使把缓冲区调到很大，传输速率仍会被慢启动的指数增长和拥塞避免的线性增长限制。这也是前端通过修改 `net.core.rmem_max` 等参数却看不到吞吐提升的常见原因。

思考题：假设一个 TCP 连接初始 cwnd=10 MSS，ssthresh=64 MSS，在进入拥塞避免后经过 3 个 RTT 出现一次超时重传。请精确计算：超时后 cwnd 和 ssthresh 分别变为多少？随后若网络恢复正常，cwnd 需要多少个 RTT 才能重新达到超时前的值？这个计算要求理解超时触发的是乘法减小到 1 MSS，而快速重传只减半，二者对恢复速度的影响截然不同，能验证是否真正理解状态机的每个分支。
