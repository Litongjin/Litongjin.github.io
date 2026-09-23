---
title: "每日基础技术总结 · 2026-09-04 · 分布式锁：Redis SETNX 与 Redlock 争议"
date: 2026-09-04 08:00:00
categories: [技术分享]
tags: ["技术分享", "分布式与架构设计"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-04 · 分布式锁：Redis SETNX 与 Redlock 争议

## 📚 今日主题

> **分布式锁：Redis SETNX 与 Redlock 争议**（分布式与架构设计）

### 1. 核心概念速览
分布式锁是解决多进程、跨网络节点并发控制原子性的基础设施。Redis SETNX 是基于单点或主从架构的非严格互斥实现，依赖 CAS（Compare-And-Swap）语义；Redlock 是由 Redis 作者提出的一种试图在多数派节点上获取强一致性锁的算法，旨在对抗网络分区下的锁失效问题。该知识点处于分布式系统 CAP 定理的工程权衡核心，对于后端高可用与数据一致性至关重要。前端工程师通常缺乏对共享状态无锁化竞争的直观体验，掌握此机制是理解分布式事务、竞态条件及最终一致性的基石。

### 2. 底层原理剖析
SETNX (Set if Not Exists) 本质是一个原子操作：检查键值是否存在，若不存在则写入并返回成功，否则失败。其线程模型对应于前端 Promise 的一次性 resolved/rejected 状态判定，但发生在服务端内存中。

Redlock 机制逻辑如下：
1. 客户端向 N 个独立的 Redis 实例发起 SETNX 请求（带 TTL），计算耗时 T3 = end_time - start_time。
2. 只有在超过半数（N/2 + 1）实例获得锁，且总耗时小于锁有效期（validity = lease_time - T3）时，才认为锁获取成功。
3. 若任一阶段失败，立即向所有已获锁的实例发送解锁指令。

对比前端概念：JavaScript 的单线程 Event Loop 保证了同一 Tick 内的状态可见性一致性；而分布式锁面对的是时钟漂移和网络抖动，类似于在前端模拟‘微任务’级别的原子性，但受限于物理世界的不可靠信道（Network Partition），导致 Redlock 在极端分区场景下仍可能出现‘双写’风险（即两个客户端同时持有有效锁）。

### 3. 基础代码与实战验证
```text
// Java 风格伪代码演示 Redlock 核心验证逻辑
int n = 5; // 实例数量
int valid_threshold = n / 2 + 1; // 多数派阈值
long validity = 10000; // 锁有效期 ms

List<String> successful_nodes = new ArrayList<>();
for (RedisClient client : clients) {
    long start = System.currentTimeMillis();
    try {
        // 原子操作：SET key value NX PX milliseconds
        boolean acquired = client.set(key, uuid, NX, EXpiry(validity)); 
        if (acquired) {
            successful_nodes.add(client.getId());
        }
    } finally {
        long end = System.currentTimeMillis();
        total_time += (end - start);
    }
}

boolean lock_acquired = false;
if (successful_nodes.size() >= valid_threshold) {
    long elapsed = System.currentTimeMillis() - start_time_of_first_request; 
    if ((elapsed + total_time) < validity) {
        lock_acquired = true;
        my_validity = validity - total_time; // 动态调整剩余有效期
    }
}

if (!lock_acquired) {
    // 释放所有已获锁的节点
    for (String nodeId : successful_nodes) {
        unlock(nodeId, key, uuid); 
    }
}
```

### 4. 常见误区与进阶思考
1. 误以为 Redlock 是强一致性方案：Redlock 无法容忍部分节点故障同时保留多数派中的错误锁，在网络脑裂（Split-Brain）场景下依然不安全，其设计初衷仅是提高可用性而非绝对正确性。
2. 忽视时钟同步对 TTL 的影响：分布式锁依赖服务器时间戳，若 NTP 同步出现较大偏差（如 > 1s），会导致锁提前过期或重复获取，必须结合单调时钟或容忍度参数设计。

思考题：如果 Redis Cluster 发生主从切换且未同步写数据时，新主节点如何处理旧主节点上的未过期锁？这如何解释为什么 Redis 官方推荐使用 Redlock 替代简单的 Zookeeper/Etcd ZAB 协议选出的单一 Master 锁？
