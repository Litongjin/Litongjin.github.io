---
title: "每日基础技术总结 · 2026-11-10 · TCP Congestion Window拥塞避免算法慢启动阈值切换"
date: 2026-11-10 07:03:20
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-10 · TCP Congestion Window拥塞避免算法慢启动阈值切换

## 📚 今日主题

> **TCP Congestion Window拥塞避免算法慢启动阈值切换**（后端基础）

### 1. 核心概念速览
### 定义

- **拥塞窗口 cwnd（Congestion Window）**：发送端维护的状态变量，表示在未收到 ACK 之前，它自认为可以注入网络的字节数上限，即允许的 **FlightSize（在途字节数）** 上限。它与接收端通告窗口共同收敛成实际发送窗口：`W = min(cwnd, rwnd)`。
- **rwnd** 反映接收端缓冲区剩余空间（流控，flow control），是**对端声明的静态背压**；**cwnd** 反映路径可用带宽与队列余量（拥塞控制，congestion control），是**发送端推断出的动态估计**。二者作用域不同：前者防止淹没接收端，后者防止淹没网络。
- **慢启动阈值 ssthresh**：一个记忆变量，保存“上次拥塞发生时路径安全水位”的估计，是 cwnd 增长律的**切换点**：
  - `cwnd < ssthresh`：慢启动，每收到一个 ACK，`cwnd += min(N, SMSS)`，等价于每个 RTT 翻倍，指数探测；
  - `cwnd >= ssthresh`：拥塞避免，每收到一个 ACK，`cwnd += SMSS*SMSS/cwnd`，等价于每个 RTT 增加约 1 个 SMSS，线性探测。

### 本质

cwnd/ssthresh 是 **AIMD（Additive Increase, Multiplicative Decrease）闭环反馈控制**在“网络不提供显式拥塞信号”这一约束下的工程实现：网络不会直接告诉发送端带宽是多少，发送端只能通过 ACK 时钟的连续性（成功）与丢包/时延抬升（失败）来逐步逼近可用容量。ssthresh 是这个逼近过程的历史记忆，避免每次丢包后都从零重新指数探测，从而把探测成本分摊到连接生命周期内。

### 在体系中的位置

位于 TCP 发送引擎核心（RFC 5681 Reno、RFC 9438 CUBIC、RFC 9293 传输层规范），向上被 HTTP/1.1、HTTP/2、gRPC、数据库驱动、Redis 客户端、MQ 生产者直接继承；向下决定网卡队列、qdisc、AQM（CoDel/FQ）的行为；横向同构扩展到 QUIC（RFC 9002 定义了与 TCP 形状一致的 cwnd/ssthresh 状态机）、SCTP、DCCP。做后端与分布式训练（AllReduce over TCP、参数服务器、RDMA 回退路径）时必须掌握，否则无法解释长肥管道上吞吐上不去、跨机房 RPC 尾延迟抖动、以及 `tcp_congestion_control` 调参的收益边界。

### 2. 底层原理剖析
### 状态机与事件驱动

发送引擎对每个事件只做三件事：更新 cwnd、更新 ssthresh、决定是否重传。

1. 初始：`cwnd = IW`（RFC 6928 建议 `10*SMSS`），`ssthresh` 初始为无穷大（Linux 实现中 `TCP_INFINITE_SSTHRESH`），因此必然先进入慢启动。
2. 新数据被 ACK：
   - `cwnd < ssthresh` → `cwnd += min(N, SMSS)`，N 为本次 ACK 新确认的字节数；
   - `cwnd >= ssthresh` → `cwnd += SMSS*SMSS/cwnd`（用累加器规避取整误差与除零）。
3. RTO 超时：`ssthresh = max(FlightSize/2, 2*SMSS)`；`cwnd = 1*SMSS`，重传最老未确认段，回到慢启动。
4. 三个重复 ACK：`ssthresh = max(FlightSize/2, 2*SMSS)`；`cwnd = ssthresh`，进入快恢复，此后每收到一个额外重复 ACK，`cwnd += SMSS` 以允许继续注入新段；收到新 ACK 后 `cwnd = ssthresh` 退出快恢复，转入拥塞避免。

### 增长律的数学本质

设窗口 W（字节），一个 RTT 内可被确认的报文段数为 W/SMSS，则每 RTT 的窗口增量为：

- 慢启动：(W/SMSS) × SMSS = W → **W 指数增长**，翻倍时间恰为一个 RTT；
- 拥塞避免：(W/SMSS) × (SMSS²/W) = SMSS → **W 线性增长**，每 RTT 只多发送一个报文段。

写成 `SMSS²/W` 并非数学游戏：它让增长速率与窗口大小成反比，从而在没有全局 RTT 计时器的前提下，纯靠 ACK 自时钟（self-clocking）精确实现“每 RTT +1 MSS”。TCP 没有定时轮询窗口的机制，一切增长由 ACK 到达驱动。

### ssthresh 的语义演进

Reno 时代 ssthresh 等价于“上次拥塞窗口的一半”，是对 BDP 的粗糙估计；一旦路径条件变化（换路由、无线随机丢包、AQM 介入），这个记忆会长期失真。

- Linux 默认 CUBIC（RFC 9438）把增长律换成以“距上次拥塞的时间”与“上次拥塞窗口”为自变量的三次函数，ssthresh 语义被弱化，主要影响慢启动退出点；
- BBR 完全抛弃丢包作为信号，用 BtlBw 与 RTprop 建模瓶颈，ssthresh 不再参与增长律；
- 但“慢启动 → 拥塞避免 → 丢包回退”的状态机骨架是所有 CC 算法的公共形状，也是理解 CUBIC/BBR 差异的基线。

### 与前端已有概念的对照

- **对比 p-limit / 信号量并发控制**：前端并发控制器是**静态整数上限**，开环、无反馈，事件源是任务完成；cwnd 是**自闭环节流器**，上限随 ACK 的成败连续变化，事件源是 ACK 时钟，控制量是字节数而非任务数。在应用层写“失败降并发、成功升并发”本质上就是简化版 AIMD，关键差异是 TCP 的乘性减基准取 FlightSize（真实在途量）而非控制器设定值。
- **对比 HTTP/1.1 每域名 6 连接**：浏览器以**连接数**表达并行度，离散、静态、无反馈；HTTP/2 多路复用把并行度收回单连接，并行度直接由 cwnd 决定——这正是“丢包链路上 HTTP/2 可能劣于 HTTP/1.1”的根本原因：帧级队头阻塞 + 所有流共享同一个 cwnd。
- **对比 TS interface 与 Java interface（类型系统延伸）**：TS 接口是编译期结构化契约、零运行时开销；Java 接口是运行期动态分派契约。同理，cwnd 是**运行期持续变化的反馈状态**，不是配置项；把 `tcp_rmem`/`tcp_wmem` 当作带宽上限来调，就是把“静态声明”误当成“动态状态”，这也是许多 K8s 网络调优方案失效的根因。

### 3. 基础代码与实战验证
```text
### 极简状态机模拟（纯标准库，无网络）

以 RTT 为步长重放 RFC 5681 增长律，观察指数→线性切换，以及丢包后 ssthresh 被重写、cwnd 塌缩的全过程。

    SMSS = 1460

    def run(rounds=20, loss_round=None):
        cwnd = 10 * SMSS          # RFC 6928 初始窗口 IW = 10*SMSS
        ssthresh = 1 << 60        # 初始视为无穷大，保证先进入慢启动
        for r in range(1, rounds + 1):
            if r == loss_round:   # 注入一次 RTO：乘性减 + 慢启动重启
                ssthresh = max(cwnd // 2, 2 * SMSS)   # 记录本次安全水位的一半，下限 2*MSS
                cwnd = 1 * SMSS
            print(f'RTT {r:2d} cwnd={cwnd/SMSS:7.2f} MSS ssthresh={ssthresh/SMSS if ssthresh < (1<<60) else float("inf"):>6}')
            acks = cwnd // SMSS   # 一个 RTT 内可被确认的段数 = 窗口大小，ACK 自时钟的本质
            if cwnd < ssthresh:
                cwnd += acks * SMSS                     # 每 ACK +1 SMSS → 每 RTT 翻倍
            else:
                cwnd += acks * (SMSS * SMSS // cwnd)    # 每 ACK +SMSS²/cwnd → 每 RTT +1 SMSS
        return cwnd

    run(rounds=20, loss_round=12)

关键行解析：

- `acks = cwnd // SMSS`：ACKE 时钟的离散化——窗口内每个报文段触发一个 ACK，因此一个 RTT 的 ACK 数正比于窗口大小。这是指数增长的来源，增长由 ACK 驱动而非定时器驱动。
- `cwnd += acks * SMSS`：慢启动的 RTT 级等价形式，第 n 个 RTT 的窗口为 `2^(n-1) * IW`；命名为“慢”仅因起点小，增长率实际最激进。
- `cwnd += acks * (SMSS*SMSS // cwnd)`：按 ACK 粒度实现时每个 ACK 只加 `SMSS²/cwnd`，乘以 ACK 数后每 RTT 净增 1 SMSS，验证线性段斜率为 1 MSS/RTT。
- `ssthresh = max(cwnd // 2, 2 * SMSS)`：丢包时以 FlightSize（此处近似等于 cwnd）的一半作为新阈值记忆，下限 2*MSS 防止窗口塌缩到无法完成重传。

### 真机验证

    sysctl net.ipv4.tcp_congestion_control                  # 确认算法，主流发行版默认 cubic
    tc qdisc add dev eth0 root netem loss 1% delay 20ms     # 注入丢包与固定 RTT
    ss -tin dst <peer_ip>                                   # 持续采样 cwnd / ssthresh
    # 输出片段：cubic ... cwnd:23 ssthresh:19 ... retrans:0/3

- 在 `ss -ti` 中持续采样 `cwnd:` 与 `ssthresh:`（部分版本字段名为 `snd_ssthresh`，Linux 内部以 MSS 为单位计 cwnd，与 RFC 的字节计量不同），可观察到：慢启动阶段 cwnd 每 RTT 翻倍；越过 ssthresh 后斜率降为 1 MSS/RTT；丢包时 ssthresh 被重写为 FlightSize/2 且 cwnd 回退。
- 用 `cwnd * MSS / rtt`（即 BDP 上界）与 `ss -ti` 中的 `bytes_acked` 对比，是排查“带宽跑不满”的第一步依据：吞吐 ≤ cwnd/RTT，若该值低于链路带宽，问题一定在窗口或 RTT，而不是应用层。
```

### 4. 常见误区与进阶思考
### 误区一：把“慢启动”理解为低速启动

命名只描述**起点小**（IW 仅 10 个 MSS），而非增长慢。慢启动是 TCP 生命周期中最激进的阶段：每 RTT 翻倍，从 10 个 MSS 涨到数千个 MSS 只需十几个 RTT。真正的风险来自这里——在未验证路径容量的情况下指数注入，突发量可能瞬间打满瓶颈队列，这也是慢启动需要 pacing、`tcp_slow_start_after_idle`、以及 CUBIC 的 capped slow start 等工程补丁的原因。

另一常被混淆的是 cwnd 与 rwnd：`W = min(cwnd, rwnd)`，若接收端 rwnd 长期偏小，cwnd 的增长只是“名义增长”，吞吐被对端背压钳制；只调大 `tcp_rmem` 而不看 cwnd 轨迹，等于在错误的变量上做优化。

### 误区二：认为丢包必然使 cwnd/ssthresh 减半

该结论只对 Reno + 三重复 ACK 的特定路径成立。实际内核中：

- RTO 超时把 `cwnd` 直接降为 `1*SMSS`（部分实现配合 PRR 渐进恢复），而非减半；
- 三重复 ACK 才进入快恢复，且 Linux 有 `tcp_cwnd_min_target` 下限，cwnd 不会无限塌缩；
- CUBIC 的乘性因子为 0.7 并带快速收敛项，ssthresh 语义已弱化；BBR 根本不以丢包为信号；
- SACK/RACK/TLP 改变了“丢包即拥塞”的判定路径，尾部丢失也不再必然触发超时重传。

因此在现代内核上直接用 Reno 公式推算线上行为，误差会很大。

### 深度思考题

设某条长肥管道上的连接长期处于 **application-limited** 状态：应用写入速率低于窗口允许的速率，FlightSize 始终小于 cwnd，网络不发满。请回答：

1. 此时 ACK 仍持续返回，cwnd 是否仍按增长律上涨？上涨出的窗口是否对应路径的真实可用容量？
2. 若应用随后突然大量写入（如大响应体开始传输），会发生什么？这与 RFC 7661（cwnd validation）试图解决的问题是什么关系？
3. 由此解释：为什么在同样 RTT、同样链路的高带宽长肥管道上，大量小请求的吞吐往往显著低于单个大请求流？请从“cwnd 能否被 ACK 时钟验证”的角度给出机制层面的回答，而不是停留在“小文件开销大”的表层结论。
