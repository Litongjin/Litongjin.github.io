---
title: "每日基础技术总结 · 2026-06-06 · TCP 快速重传与选择性确认 SACK"
date: 2026-06-06 08:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-06-06 · TCP 快速重传与选择性确认 SACK

## 📚 今日主题

> **TCP 快速重传与选择性确认 SACK**（网络基础）

### 1. 核心概念速览
TCP快速重传（Fast Retransmit）与选择性确认（SACK）是传输层可靠传输机制中的两个关键算法/选项。快速重传在接收方收到乱序数据时立即发送重复ACK，发送方在收到3个重复ACK后不等待RTO超时即重传疑似丢失的报文段，从而降低重传延迟。SACK是一个TCP选项，允许接收方在ACK中携带已经收到的非连续数据块的边界信息，使发送方能够精确重传真正丢失的段，避免传统Go-Back-N方式下重传所有后续未确认数据造成的带宽浪费。它们在TCP/IP协议栈中位于传输层，是TCP实现高效可靠传输的基石。专业工程师必须掌握，因为其核心思想——通过反馈信息精确定位缺失并避免全量重试——广泛应用于分布式系统、HTTP/2、QUIC、前端资源加载等场景，理解它有助于设计高可靠低延迟的网络应用。

### 2. 底层原理剖析
快速重传机制：接收方每收到一个乱序报文段，就立即生成一个重复ACK，ACK序号为期望收到的下一个序号。发送方当收到3个重复ACK（即总共4个相同ACK）时，认为该序号的数据丢失，立即重传，无需等待RTO。注意：重复ACK可能因报文段重排序产生，因此需要“3个重复ACK”阈值来避免误判。
SACK机制：通过TCP头部的SACK选项，接收方可以告诉发送方自己已接收的不连续数据块，每个块用(Left Edge, Right Edge)表示，Left Edge为第一个已收序号，Right Edge为最后一个已收序号+1（即不包含）。发送方根据SACK信息维护一个“丢失队列”和“已接收队列”，只重传丢失的段。
工作流程：1. 发送方发送序列数据；2. 接收方收到乱序数据，缓存并发送SACK选项；3. 发送方收到重复ACK并携带SACK，分析哪些段丢失，立即重传这些段，同时可继续发送新数据。
对比前端概念：类似React的虚拟DOM diff只更新变化节点，而非全量重绘；也类似于HTTP/2的流级重传。与Java/TS接口区别类比：Java接口是显式声明并编译期检查，TS接口是结构化类型，运行时不保留；同理，快速重传是TCP默认行为，SACK是可选选项，需握手时协商（双方在SYN中携带SACK-permitted选项），且若对端不支持则回退到基础快速重传。这样理解可以更清晰地区分协议的核心机制与增强特性。

### 3. 基础代码与实战验证
```text
接收方逻辑伪代码：
维护期望接收序号 expected_seq
当收到TCP段 (seq, len)：
  if seq == expected_seq：
    正常处理，更新 expected_seq = seq + len
    发送 ACK(ack = expected_seq)
  else if seq > expected_seq：
    缓存该段，记录其边界 [seq, seq+len)
    发送 ACK(ack = expected_seq)  // 重复ACK
    同时生成SACK选项，携带所有已缓存乱序段的边界

发送方逻辑伪代码：
维护已发送未确认的段集合 outstanding
维护 last_ack 和 dupack_count
对每个收到的ACK(ack, sack_blocks)：
  if ack > last_ack：
    last_ack = ack
    dupack_count = 0
    删除 outstanding 中序号小于 ack 的段
  else if ack == last_ack：
    dupack_count++
    if dupack_count == 3：
      if sack_blocks 存在：
        根据 sack_blocks 和 outstanding 推断丢失段（即 outstanding 中未被任何 SACK 块覆盖且序号小于 last_ack + 窗口内未确认的段）
        重传这些丢失段
      else：
        重传序号为 ack 的段（即 Go-Back-N）
      dupack_count = 0  // 或重置为阈值以下，具体实现而定

关键点：SACK 块只告诉发送方哪些数据已收到，发送方通过“已被确认但未在SACK中出现”来推断丢失。实际实现需处理SACK块的合并和重复ACK的边界情况。
```

### 4. 常见误区与进阶思考
误区1：认为快速重传是超时后的机制。实际上快速重传是在RTO超时之前，基于重复ACK触发的提前重传，目的是缩短丢包恢复时间。若没有重复ACK，仍会等待超时。
误区2：认为SACK能完全避免不必要的重传。SACK只提供接收方的接收状态，发送方仍需推断丢失段；并且SACK选项长度有限，当乱序块过多时，只能部分告知。此外，SACK本身不触发重传，它只是辅助快速重传选择正确报文段。
思考题：如果接收方收到大量乱序数据，且每个重复ACK都携带了不同的SACK信息，发送方如何区分“真正的丢包”和“网络重排序”？请描述发送方依据重复ACK计数和SACK块的具体决策逻辑，以及为什么需要连续3个重复ACK而不是1个？
