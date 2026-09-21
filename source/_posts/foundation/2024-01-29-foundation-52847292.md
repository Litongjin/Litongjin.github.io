---
title: "每日基础技术总结 · 2024-01-29 · 负载均衡最少连接与加权轮询算法"
date: 2024-01-29 20:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-01-29 · 负载均衡最少连接与加权轮询算法

## 📚 今日主题

> **负载均衡最少连接与加权轮询算法**（网络基础）

### 1. 核心概念速览
负载均衡算法旨在将客户端请求分发至后端服务器集群，以优化资源利用率、最大化吞吐量并最小化响应时间。最少连接数（Least Connections, LC）是一种动态加权算法，其核心机制是统计当前活跃连接数，优先将新请求分配给负载最低的节点，适用于长连接或处理耗时差异巨大的场景。加权轮询（Weighted Round Robin, WRR）是一种静态/准静态算法，按预设权重比例有序分配请求，忽略当前实时负载，适用于无状态且处理能力均匀的短连接场景。在计算机体系结构中，位于应用层（L7）与传输层（L4）之间，是分布式系统高可用架构的关键组件。专业工程师必须掌握它，因为错误选型会导致负载不均、雪崩效应或性能瓶颈，直接影响系统的 SLA。

### 2. 底层原理剖析
1. 加权轮询 (WRR) 机制：维护一个虚拟的环形队列或计数器，每个服务器拥有固定的权重值。每次选择时，遍历所有存活服务器，选取权重最高者或累加器溢出者为下一节点，随后减去该权重。若服务器故障，则从序列中移除。
2. 最少连接 (LC) 机制：维护全局或局域的活跃连接计数器（Active Connections）。当新请求到达时，遍历服务器列表，计算 metric = current_connections / capacity（或直接比较 raw connections），选择 metric 最小值对应的节点。连接建立时计数+1，断开时-1。
对比前端概念：这类似于 TypeScript 中的 'Interface' (WRR，定义结构化的固定契约/比例) 与 'Union Type + Runtime Check' (LC，基于运行时状态的动态类型判断)。WRR 类似编译时的静态检查，规则固定；LC 类似运行时的鸭子测试，根据实例实际占用资源动态决策。前者注重公平性和可预测性，后者注重实时的效率最优。

### 3. 基础代码与实战验证
```text
// 简化版加权轮询实现逻辑
let serverList = [
  { id: 1, weight: 5 },
  { id: 2, weight: 3 }
];
let currentIndex = 0;
let currentWeight = 0;

function getNextServer() {
  // 1. 找到最大权重
  let maxWeight = Math.max(...serverList.map(s => s.weight));
  // 2. 遍历更新当前累计权重
  for (let i = 0; i < serverList.length; i++) {
    if (serverList[i].weight > 0) {
      currentWeight += serverList[i].weight;
      // 3. 如果当前累计权重大于等于最大权重，选中该节点
      if (currentWeight >= maxWeight) {
        currentIndex = i;
        break;
      }
    }
  }
  // 4. 重置累计权重为最大权重的负值（平滑加权轮询核心修正）
  currentWeight -= maxWeight;
  return serverList[currentIndex];
}

// 最少连接伪代码逻辑：
/*
function getLeastConnectedNode(nodes) {
  let minConn = Infinity;
  let targetNode = null;
  for (node in nodes) {
    if (node.activeConnections < minConn) {
      minConn = node.activeConnections;
      targetNode = node;
    }
  }
  return targetNode;
} // 注：实际生产中需考虑并发锁保证原子性 */
```

### 4. 常见误区与进阶思考
误区一：认为最少连接数总是优于轮询。实际上，LC 算法在应对突发性批量请求时可能引发‘惊群效应’或导致部分节点长期闲置，且在无状态服务中增加连接计数的开销（Context Switch 和 Memory Access）可能抵消其收益。对于纯 HTTP 短连接，WRR 通常更稳定且 CPU 开销更低。

误区二：混淆权重配置与实际负载能力。权重应基于服务器的硬件规格（CPU/RAM）和业务特性配置，而非简单人为指定。若未进行健康检查剔除失效节点，WRR 会将流量持续分发到宕机节点，LC 会将请求分发到假死但心跳尚存的节点，导致整体成功率下降。

深度思考题：在一个包含 100 个节点的微服务集群中，如果其中 10% 的节点突然因 GC 停顿导致响应延迟增加 10 倍，而剩余节点正常运行，请问加权轮询和最少连接算法分别会产生怎样的系统性后果？如何结合 TCP Keepalive 或 HTTP Health Check 机制来缓解这一问题？
