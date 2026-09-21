---
title: "每日基础技术总结 · 2025-12-06 · TCP 的快速重传与 SACK 选项"
date: 2025-12-06 20:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2025-12-06 · TCP 的快速重传与 SACK 选项

## 📚 今日主题

> **TCP 的快速重传与 SACK 选项**（网络基础）

### 1. 核心概念速览
快速重传（Fast Retransmit）与选择确认（SACK, Selective Acknowledgment）是 TCP 传输层为克服传统累计确认（Cumulative ACK）局限性而设计的两项关键拥塞控制与可靠性增强机制。传统机制下，单个报文段丢失会导致后续所有有序报文被丢弃且接收方仅回复重复的 ACK，导致发送方陷入超时等待才能触发重传，极大地降低了网络利用率并加剧了队头阻塞（Head-of-Line Blocking）。快速重传通过检测连续三个重复 ACK 推断出报文段丢失而非网络延迟，立即重传该特定报文；SACK 则允许接收方在 ACK 中明确告知发送方已成功接收的非连续数据块范围，从而避免发送方盲目重传已到达的数据或无效探测。这两者共同构成了现代高性能后端服务处理高丢包率场景、保障数据一致性与吞吐量的基石，也是 AI 大规模分布式训练（如参数服务器通信）中优化长尾延迟的核心理论基础。

### 2. 底层原理剖析
核心逻辑在于从『被动等待超时』转变为『基于信号推断的主动重传』与『基于状态同步的精准定位』。

1. 快速重传判定条件：
   接收端收到失序报文时，不丢弃而是缓存并发送针对之前最后一个有序序列号的重复 ACK。当发送端连续接收到 3 个相同的重复 ACK（DupACKs=3），即意味着中间必然有数据丢失，无需等待 RTO（重传定时器）超时，立即执行重传。

2. SACK 数据结构交互：
   SACK 选项插入到 TCP 头部选项中（Option Kind=4）。包含至少一个 'Start' 和 'End' 字段对，描述接收窗口内已正确接收但尚未被累积 ACK 覆盖的数据区间。例如 SACK: 1001-2000, 3001-4000。

3. 前端知识体系对比 - TS Interface vs Java Interface:
   这类似于 TypeScript 的结构类型系统（Structural Typing）与 Java 的名义类型系统（Nominal Typing）的区别，或者更贴切地说是『契约』的不同实现方式。
   - 传统 TCP ACK (Cumulative) 类似严格的命名接口：必须按顺序完美兑现所有方法调用，任何一次调用失败（丢包），整个后续流程被视为未定义状态，直到强制重置连接或抛出异常（Timeout/Retransmission）。
   - SACK 类似灵活的结构体验证：它允许部分成功。即使某些方法调用乱序或部分缺失（网络抖动），接收方可以明确指出哪些结构特征（数据块）已经匹配成功。这种‘局部成功’的信息反馈使得发送方可以快速修补缺失部分，而不需要重新执行整个对象创建过程。它将『全有或全无』的强一致性约束解耦为『部分提交、局部修复』的最终一致性辅助手段。

### 3. 基础代码与实战验证
```text
// 模拟 TCP SACK 处理的伪代码逻辑
// 关注点：如何解析 SACK 选项并指导重传决策

class TCPSocket {
    constructor() {
        this.nextSeq = 1;      // 下一个要发送的序列号
        this.unackedData = {}; // 已发送但未获确认的数据缓冲区
        this.receivedSacks = []; // 最近收到的 SACK 列表
    }

    onReceivePacket(packet) {
        if (packet.type === 'ACK') {
            const sackOptions = parseTCPOptions(packet.options);
            
            // 更新累积 ACK
            const newCumAck = packet.ackNumber;
            this.confirmUpTo(newCumAck);

            // 处理 SACK：标记非连续已接收区域
            for (const sackBlock of sackOptions.SACKBlocks) {
                const { start, end } = sackBlock;
                // 将这些范围标记为有效，避免后续错误重传
                this.markConfirmed(start, end);
                this.receivedSacks.push(sackBlock); 
            }
        } 
        else if (packet.type === 'DUP_ACK') {
            // 快速重传机制入口
            if (++this.dupAckCount >= 3 && !this.fastResendTriggered) {
                // 推断出 dupAckNumber 之后的字节丢失
                const lostSeq = packet.dupAckNumber + 1;
                this.resend(lostSeq); // 直接重传特定序列号，而非整个窗口
                this.fastResendTriggered = true;
                this.dupAckCount = 0;
            }
        }
    }

    // 利用 SACK 信息优化重传策略
    optimizeRetransmission() {
        // 结合 SACK 和 DupACKs 确定最可能的丢失范围
        // 如果 SACK 显示 [100-200] 已收，但 CumAck 停在 90，说明 91-100 之间有问题
        const confirmedRanges = this.receivedSacks;
        const lastCumAck = this.getLastCumulativeAck();
        
        for (const range of confirmedRanges) {
            if (range.start > lastCumAck) {
                // 计算 gap，仅重传 gap 部分
                this.resendGap(lastCumAck, range.start);
            }
        }
    }
}
```

### 4. 常见误区与进阶思考
误区一：认为 SACK 能完全消除重传。SACK 只是提高了重传的准确性，它无法解决因为真正丢包导致的延迟问题，也不能替代丢包检测算法（如 BBR/CUBIC）。如果在极度恶劣的网络环境下，SACK 可能会导致发送方过度响应频繁的重传请求，反而引发新的拥塞。

误区二：混淆『快速重传』与『选择性重传』。快速重传是事件触发机制（Event-driven），由 DupACK 计数触发；选择性重传是调度策略。虽然两者常配合使用，但在某些极端情况下，快速重传可能误判（如乱序而非丢包），此时若无 SACK 辅助，重传无效数据会浪费带宽。

思考题：假设一个 TCP 连接在局域网环境中发生了一次轻微的乱序（Out-of-order，非丢包），快速重传机制被错误触发并重传了一个实际上已经到达的数据包。请分析在存在 SACK 选项的情况下，接收端的协议栈是如何识别这一冗余重传并防止数据重复或死锁的？这与无 SACK 的传统 TCP 处理有何本质区别？
