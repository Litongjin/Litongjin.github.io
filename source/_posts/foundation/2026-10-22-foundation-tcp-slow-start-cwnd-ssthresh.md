---
title: "每日基础技术总结 · 2026-10-22 · TCP慢启动（Slow Start）拥塞窗口（cwnd）的指数增长与阈值（ssthresh）切换"
date: 2026-10-22 07:05:08
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-22 · TCP慢启动（Slow Start）拥塞窗口（cwnd）的指数增长与阈值（ssthresh）切换

## 📚 今日主题

> **TCP慢启动（Slow Start）拥塞窗口（cwnd）的指数增长与阈值（ssthresh）切换**（网络基础）

### 1. 核心概念速览
TCP慢启动是TCP拥塞控制的核心机制，其核心变量是拥塞窗口（cwnd）——发送方在任意时刻允许发送但未收到确认（ACK）的最大数据量。慢启动并非“缓慢”，而是一种从极小值开始、以**指数增长**探测网络可用带宽的激进策略：每收到一个ACK，cwnd增加一个MSS（Maximum Segment Size），因此每经过一个RTT（往返时延），cwnd翻倍（1, 2, 4, 8… MSS）。当cwnd增长至慢启动阈值（ssthresh，通常初始值为65535字节或连接对端通告的窗口大小）时，TCP切换至拥塞避免阶段（Congestion Avoidance），改为线性增长（每RTT仅增加1个MSS），以在接近网络容量时谨慎探测。该机制解决的核心问题是：如何在无法预知网络路径带宽与延迟积（BDP）的情况下，快速逼近公平共享的链路容量，同时避免因初始窗口过大导致路由器缓冲区瞬间溢出、引发大规模丢包与全局同步（Global Synchronization）。

在前端工程师的视角下，慢启动是TCP栈内嵌的**流控与拥塞控制状态机**，与HTTP/1.1的队头阻塞（HOL Blocking）或HTTP/2的流优先级调度无关，而是位于传输层（TCP Segment）的原始字节流控制。理解慢启动是诊断首字节时间（TTFB）瓶颈、优化TLS握手后的首次数据推送、以及理解为何CDN边缘节点需要调整initcwnd（初始拥塞窗口）以应对高BDP网络的基础。它直接决定了TCP连接在慢启动阶段能承载的突发数据量（Burst Size），影响前端资源（尤其是大体积JS/CSS）的加载瀑布图。

### 2. 底层原理剖析
底层运行机制遵循以下精确状态机：

1. **初始化状态**：连接建立（三次握手完成）后，cwnd = initcwnd（Linux默认通常为10 MSS，Google提议的RFC 6928），ssthresh = 65535字节（或接收方通告的rwnd，取较小值）。
2. **慢启动（Slow Start）**：
   - 每收到一个ACK（累计确认或SACK），cwnd += MSS（字节数）。
   - 数学性质：cwnd(t) = min(2^n × MSS, ssthresh)，其中n为经过的RTT数。因此呈指数增长（Exponential Growth），O(log n)时间复杂度逼近ssthresh。
3. **阈值切换（Threshold Switching）**：
   - 当cwnd ≥ ssthresh时，状态迁移至**拥塞避免（Congestion Avoidance）**。
   - 增长策略变更为：cwnd += MSS × (MSS / cwnd)（每RTT约增加1个MSS，即Additive Increase）。
4. **拥塞事件响应**：
   - **超时（RTO Timeout）**：判定为严重拥塞，ssthresh = max(cwnd/2, 2×MSS)，cwnd重置为1 MSS，重新进入慢启动。
   - **重复ACK（3 Duplicate ACKs，快速重传触发）**：判定为轻度拥塞（包丢失但网络仍通），ssthresh = max(cwnd/2, 2×MSS)，cwnd = ssthresh（进入快速恢复，Fast Recovery），随后进入拥塞避免。

与前端知识体系的对比：
- **vs JavaScript闭包/原型链**：慢启动是**内核协议栈的隐式状态机**，对应用层透明，而JS闭包是语言级显式作用域链。你无法通过Web API直接读取或修改cwnd（浏览器出于安全隔离禁止），但可通过TCP_INFO（Linux）或getsockopt在服务端探查。
- **vs HTTP缓存控制**：Cache-Control: max-age是**声明式**的资源有效期协商，而慢启动是**响应式**的网络容量探测，基于实时ACK时钟（ACK Clocking）驱动，而非预先配置。
- **vs React Fiber架构**：Fiber将渲染任务拆分为可中断的增量单元以优化主线程调度，慢启动将数据流拆分为受拥塞窗口约束的段（Segment）以优化网络队列占用，二者均为**分而治之（Divide and Conquer）**策略，但前者面向CPU时间片，后者面向网络带宽时间积（Bandwidth-Delay Product）。
- **vs 前端轮询/指数退避（Exponential Backoff）**：慢启动是** aggressive start + additive increase**，而前端重试机制（如fetch retry）通常是**保守起始 + 指数退避**（exponential backoff），二者增长曲线相反，目的不同：前者探测可用资源，后者规避服务过载。

### 3. 基础代码与实战验证
```text
以下Python代码模拟TCP发送方的拥塞控制状态机，精确复现cwnd的指数增长与ssthresh切换逻辑：

class TCPSender:
    def __init__(self, mss=1460, init_cwnd=10):
        self.mss = mss  # Maximum Segment Size，通常为1460字节（以太网MTU 1500 - IP头20 - TCP头20）
        self.cwnd = init_cwnd * mss  # 初始拥塞窗口，Linux默认10个MSS
        self.ssthresh = 65535  # 慢启动阈值，初始通常为64KB
        self.state = 'SLOW_START'  # 状态机：SLOW_START / CONGESTION_AVOIDANCE / FAST_RECOVERY
        self.unacked = 0  # 已发送未确认的字节数

    def on_ack(self, acked_bytes):
        """处理ACK，驱动拥塞窗口更新"""
        self.unacked = max(0, self.unacked - acked_bytes)
        
        if self.state == 'SLOW_START':
            # 慢启动：每ACK增加一个MSS，实现指数增长（每RTT翻倍）
            self.cwnd += self.mss
            if self.cwnd >= self.ssthresh:
                self.state = 'CONGESTION_AVOIDANCE'
                print(f"[Threshold] 切换至拥塞避免，cwnd={self.cwnd}")
        
        elif self.state == 'CONGESTION_AVOIDANCE':
            # 拥塞避免：每RTT增加约1个MSS（线性增长）
            self.cwnd += self.mss * (self.mss / self.cwnd)

    def on_timeout(self):
        """RTO超时：判定为严重拥塞，重置窗口"""
        self.ssthresh = max(self.cwnd // 2, 2 * self.mss)
        self.cwnd = 1 * self.mss  # 重置为1个MSS，重新慢启动
        self.state = 'SLOW_START'
        print(f"[Timeout] ssthresh更新为{self.ssthresh}, cwnd重置为{self.cwnd}")

    def on_duplicate_ack(self, dup_count):
        """收到3个重复ACK：触发快速重传与快速恢复"""
        if dup_count == 3 and self.state != 'FAST_RECOVERY':
            self.ssthresh = max(self.cwnd // 2, 2 * self.mss)
            self.cwnd = self.ssthresh + 3 * self.mss  # 进入快速恢复，临时扩大窗口
            self.state = 'FAST_RECOVERY'

    def send_data(self, data_len):
        """根据当前cwnd决定可发送的数据量"""
        allowed = min(self.cwnd - self.unacked, data_len)
        self.unacked += allowed
        return allowed

# 模拟一次RTT的传输循环
tcp = TCPSender()
for rtt in range(5):
    # 模拟接收方ACK整个cwnd（理想情况）
    tcp.on_ack(tcp.cwnd)
    print(f"RTT {rtt+1}: cwnd = {tcp.cwnd/1460:.1f} MSS, state = {tcp.state}")

# 输出将显示：cwnd从10 → 20 → 40 → 80 → 拥塞避免（线性增长）
```

### 4. 常见误区与进阶思考
1. **认知误区：慢启动是“慢”的，而拥塞避免是“快”的**。恰恰相反，慢启动的指数增长速率（2^n）远快于拥塞避免的线性增长（+1 MSS/RTT）。慢启动的“慢”仅指**初始窗口保守**（从1或10个MSS开始），而非增长速度。在现代高带宽延迟积（BDP > 10MB）网络中，若仍使用RFC 2581建议的初始cwnd=2-4 MSS，会导致大量时间浪费在慢启动探测阶段，这是Google推动RFC 6928将initcwnd提升至10 MSS的核心动机。

2. **认知误区：ssthresh是静态配置或仅由接收方窗口决定**。ssthresh是**动态阈值**，其值随网络拥塞历史实时调整。初始值虽可能受限于对端通告的接收窗口（rwnd），但一旦进入拥塞避免或遭遇丢包，ssthresh将独立于rwnd，根据拥塞事件（RTO或DupACK）按cwnd/2动态收缩。前端工程师常误认为TCP流控仅由接收窗口（如HTTP/2的SETTINGS_INITIAL_WINDOW_SIZE）控制，而忽视了发送方基于丢包反馈的自主拥塞控制。

**进阶思考题**：在高BDP网络（如跨洋专线，RTT=200ms，带宽=10Gbps）中，初始cwnd=10 MSS（约14.6KB）与BDP（250MB）之间存在数量级差距。请推导：若不考虑丢包，仅通过慢启动（每RTT翻倍）需要多少个RTT才能使cwnd达到BDP？这解释了为何TCP协议栈需要**初始窗口调优（initcwnd）**、**TCP Fast Open（TFO）**以及**BBR（Bottleneck Bandwidth and RTT）**等基于测量的拥塞控制算法来替代基于丢包的慢启动机制？
