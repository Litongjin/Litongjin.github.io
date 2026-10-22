---
title: "每日基础技术总结 · 2026-10-23 · TCP快速重传（Fast Retransmit）与快速恢复（Fast Recovery）的状态机行为"
date: 2026-10-23 07:02:06
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-23 · TCP快速重传（Fast Retransmit）与快速恢复（Fast Recovery）的状态机行为

## 📚 今日主题

> **TCP快速重传（Fast Retransmit）与快速恢复（Fast Recovery）的状态机行为**（网络基础）

### 1. 核心概念速览
TCP快速重传（Fast Retransmit）与快速恢复（Fast Recovery）是TCP拥塞控制机制中处理丢包的两个协同算法。快速重传解决『RTO（Retransmission Timeout）超时等待过长』的问题：接收方收到乱序段后立即发送重复ACK（duplicate ACK），发送方收到3个重复ACK时不等待RTO超时就立即重传丢失段。快速恢复解决『重传后窗口重置导致吞吐量骤降』的问题：触发快速重传后，发送方不进入慢启动（Slow Start），而是执行拥塞避免（Congestion Avoidance）的变体——将慢启动阈值ssthresh设为当前拥塞窗口cwnd的一半，将cwnd设为ssthresh+3（补偿3个重复ACK代表的已离开网络的段），之后每收到一个重复ACK就将cwnd加1（窗口膨胀），收到确认新数据的ACK时将cwnd缩减回ssthresh并退出快速恢复。这两个机制使TCP在轻度丢包（单段丢失）场景下避免RTO的指数退避惩罚和慢启动的窗口归零重建，将吞吐量损失从秒级压缩到RTT级。位置：传输层核心可靠性机制，位于TCP协议栈内核态实现（如Linux的tcp_input.c），是理解TCP性能调优（如net.ipv4.tcp_congestion_control、BBR算法对比）、网络编程中write/recv行为、抓包分析（Wireshark中的TCP Fast Retransmission标记）的必备基础。前端工程师必须掌握它的原因：HTTP/1.1和HTTP/2均构建于TCP之上，页面加载性能、API响应延迟直接受TCP重传行为影响；当出现『接口偶发性卡顿』『大文件上传中断』等问题时，快速重传与快速恢复的状态转换是定位问题的关键线索。

### 2. 底层原理剖析
状态机转换逻辑（发送方视角）：

1. 正常状态（Open）：cwnd按拥塞避免（每RTT加1 MSS）或慢启动（每RTT翻倍）增长。ssthresh初始为65535字节。

2. 触发快速重传：
   - 条件：收到3个重复ACK（即收到4个相同ACK号的ACK包，第一个为正常ACK，后续3个为duplicate）
   - 动作：
     a) 立即重传丢失的段（无需等待RTO计时器到期）
     b) ssthresh = max(cwnd / 2, 2 * MSS)
     c) cwnd = ssthresh + 3 * MSS（+3补偿3个已确认离开网络的段）
     d) 进入快速恢复状态

3. 快速恢复状态：
   - 每收到1个重复ACK：cwnd += MSS（窗口膨胀，允许发送新数据，因为每个重复ACK代表一个段已离开网络到达接收方）
   - 收到确认新数据的ACK（即ACK号大于重传前的最高ACK号）：
     a) cwnd = ssthresh（窗口缩减回新阈值，称为『窗口收缩』）
     b) 退出快速恢复，回到正常状态（Open），进入拥塞避免阶段
   - 若RTO超时（极端情况）：cwnd = 1 MSS，ssthresh = max(cwnd / 2, 2 * MSS)，进入慢启动

4. 伪代码（简化版）：
   on_ack_received(ack_num):
     if ack_num == last_ack_num:
       dup_ack_count++
       if dup_ack_count == 3 and state == OPEN:
         retransmit_lost_segment()
         ssthresh = max(cwnd / 2, 2 * MSS)
         cwnd = ssthresh + 3 * MSS
         state = FAST_RECOVERY
       elif state == FAST_RECOVERY:
         cwnd += MSS  // 窗口膨胀
     else:  // 收到新ACK
       if state == FAST_RECOVERY:
         cwnd = ssthresh  // 窗口收缩
         state = OPEN
       last_ack_num = ack_num
       dup_ack_count = 0

与前端概念的对比：
- vs Promise重试：Promise重试是应用层逻辑，超时时间由开发者设定，重试次数固定；快速重传是传输层自动机制，由ACK模式触发，无需应用介入。
- vs RxJS的retryWhen操作符：retryWhen基于流的事件驱动，可自定义退避策略；快速恢复是内核态状态机，策略固定（RFC 2001/2581/5681），不可编程。
- vs HTTP缓存：HTTP缓存减少请求次数，优化带宽；快速重传减少重传延迟，优化单连接吞吐量。
- 与HTTP/2的关联：HTTP/2多路复用单个TCP连接，若TCP层触发快速重传，所有流（stream）都会被阻塞（TCP层head-of-line blocking），这是HTTP/3转向QUIC的核心动机之一。

关键细节：
- 快速重传只能处理『单个段丢失』。若同一窗口内多个段丢失，快速恢复可能失败，仍需RTO。
- SACK（Selective Acknowledgment）扩展可告知发送方具体哪些段已收到，允许在快速恢复中重传多个丢失段（NewReno算法，RFC 6582）。
- Linux实现中，可通过net.ipv4.tcp_early_retrans参数调整触发阈值（默认3，可改为更早触发）。

### 3. 基础代码与实战验证
```text
验证方法：使用tcpdump或Wireshark抓包观察快速重传行为。

步骤1：模拟丢包环境（Linux + tc命令）
  # 在发送方机器上添加10ms延迟和1%丢包率
  sudo tc qdisc add dev eth0 root netem delay 10ms loss 1%

步骤2：启动一个TCP服务并传输数据
  # 服务端：nc -l 9999 > received_file
  # 客户端：cat large_file | nc server_ip 9999

步骤3：抓包观察
  sudo tcpdump -i eth0 -w capture.pcap 'tcp port 9999'

步骤4：用Wireshark打开capture.pcap，筛选器输入：
  tcp.analysis.fast_retransmission

预期观察到的包序列（简化）：
  Seq=1000, Len=1460  [原始发送]
  Seq=2460, Len=1460  [原始发送]
  Seq=3920, Len=1460  [丢失，网络中未到达]
  Seq=5380, Len=1460  [原始发送]
  Ack=3920            [接收方：我期望3920，但收到了5380，发重复ACK]
  Ack=3920            [dup ACK #1]
  Ack=3920            [dup ACK #2]
  Ack=3920            [dup ACK #3，触发快速重传]
  Seq=3920, Len=1460  [快速重传：不等RTO，立即重发3920段]
  Ack=6840            [接收方：3920和后续都收到了，确认到6840]
  [此时cwnd从ssthresh+3收缩回ssthresh，进入拥塞避免]

通过/proc/net/tcp观察内核状态（需root）：
  # 查看当前TCP连接的拥塞窗口和状态
  cat /proc/net/tcp | grep ':270F'  # 270F是9999端口的16进制
  # 输出第4列（state）含义：
  #   01 = ESTABLISHED（正常）
  #   状态字段不直接显示快速恢复，但可通过ss -i命令查看：
  ss -i dst server_ip
  # 输出示例：
  #   cubic wscale:7,7 rto:204 rtt:2.5/1.5 cwnd:10 ssthresh:7
  #   若cwnd骤降且ssthresh减半，说明刚经历快速恢复

清除模拟环境：
  sudo tc qdisc del dev eth0 root
```

### 4. 常见误区与进阶思考
误区1：认为快速重传可以替代RTO，RTO机制可以被禁用。
真相：快速重传仅适用于『收到3个重复ACK』的场景，前提是网络中只有单个段丢失且后续段能到达接收方触发重复ACK。若发生以下情况，RTO仍是唯一保障：(a) 整个窗口的数据全部丢失，接收方无任何段可确认；(b) ACK包本身丢失；(c) 窗口大小小于4个段（无法产生3个重复ACK）。Linux内核中RTO最小值为200ms（tcp_rto_min），是最后的兜底机制。实际抓包中会看到『Fast Retransmission』和『Retransmission (RTO)』并存。

误区2：认为快速恢复后cwnd回到丢包前的水平，吞吐量无损。
真相：快速恢复将ssthresh降为cwnd/2（乘法减小，multiplicative decrease），然后cwnd从ssthresh开始线性增长（加法增大，additive increase）。这意味着每次丢包事件后，cwnd会经历『腰斩→缓慢恢复』的过程，长期吞吐量为sawtooth（锯齿）波形。假设丢包率1%，稳定状态下cwnd在W/2到W之间波动，平均吞吐量为(3/4)*W/RTT。这是TCP Reno/NewReno的固有特性，也是BBR等现代算法试图改进的点（BBR不基于丢包，基于带宽和RTT测量）。

思考题：为什么快速恢复中每收到一个重复ACK要将cwnd加1 MSS（窗口膨胀），而不是保持cwnd不变？
提示：从『ACK时钟』（ACK clocking）角度思考——TCP的自时钟（self-clocking）机制依赖ACK到达速率来控制发送速率。每个重复ACK代表网络中有一个段已到达接收方并离开网络，此时网络容量有空余，允许发送方注入一个新段。若不膨胀窗口，发送方会因窗口耗尽而暂停发送，浪费了网络已恢复的空闲容量。
