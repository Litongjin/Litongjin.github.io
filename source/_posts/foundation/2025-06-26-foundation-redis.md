---
title: "每日基础技术总结 · 2025-06-26 · Redis 集群：哈希槽与故障转移"
date: 2025-06-26 20:00:00
categories: [技术分享]
tags: ["技术分享", "数据库与缓存进阶"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2025-06-26 · Redis 集群：哈希槽与故障转移

## 📚 今日主题

> **Redis 集群：哈希槽与故障转移**（数据库与缓存进阶）

### 1. 核心概念速览
Redis Cluster 采用去中心化的分片架构，核心机制为 CRC16 哈希映射至 16384 个预分配的哈希槽（Hash Slots）。该设计解决了传统主从复制中单点故障与容量瓶颈问题，通过数据分区实现水平扩展。其本质是将键空间离散化并动态分配给多个 Master 节点，每个节点独立负责特定槽位范围内的读写。在 AI/大数据体系中，它是构建高吞吐、低延迟分布式缓存层的基础组件，掌握它意味着理解分布式一致性hash、故障域隔离及在线重平衡的底层逻辑，这是从单体应用迈向大规模微服务架构的必经之路。

2. 底层原理剖析：
数据定位算法：对 Key 执行 CRC16(Key) % 16384 计算得出槽位索引。客户端直接根据槽位连接对应的 Master 节点；若错误连接至持有该槽位的其他节点，服务端返回 -MOVED <slot> <host:port> 引导请求跳转。
故障转移机制（Failover）：基于 Raft 协议变体。当集群检测到 Master 心跳超时（默认 failover-timeout），属于该 Master 的所有 Slave 节点发起选举。获得多数派（Quorum）Slave 支持的节点升级为新的 Master，接管所有原 Master 负责的哈希槽。随后，新 Master 向集群发送 CLUSTERSETCONFIGEPOCH，同步槽位所有权变更，并完成全量或增量数据同步（PSYNC）。此过程对客户端透明，仅表现为短暂的重定向。
前端对比：这类似于 TypeScript 中的接口契约与运行时校验。TS 接口仅在编译期定义类型结构（如同哈希槽的定义范围），而运行时 Redis 节点的行为（如处理 MOVED 异常）则像 JS 的运行时错误抛出与捕获。前端的 'Props Drilling' 或 Context API 传递状态，类似这里客户端缓存 Slot-to-Node 映射关系，但若映射过期（元数据不同步），必须像重试机制一样处理 MOVED 异常重新获取最新拓扑。

### 3. 基础代码与实战验证
```text
// Node.js 原生 Net Socket 模拟 Redis 集群故障转移后的 MOVED 重定向处理
// 注意：真实生产环境通常使用 ioredis 等库，此处仅演示底层协议交互逻辑
const net = require('net');

async function executeWithRedirect(host, port, command) {
    let currentHost = host;
    let currentPort = port;

    // 无限循环直到成功或达到最大重试次数（防止死循环）
    for (let attempt = 0; attempt < 5; attempt++) {
        const client = new net.Socket();
        await connect(client, currentHost, currentPort);

        // 发送命令
        const response = await sendCommand(client, command); 
        client.destroy();

        // 检查是否为 MOVED 错误，例如: -MOVED 3999 127.0.0.1:6381
        if (response.startsWith('-MOVED')) {
            const parts = response.split(' ');
            // 解析目标节点 IP 和端口，更新本地映射状态
            currentHost = parts[2].split(':')[0];
            currentPort = parseInt(parts[2].split(':')[1], 10);
            console.log(`MOVED: Re-routing to ${currentHost}:${currentPort}`);
            continue; // 重试连接新节点
        }

        return response; // 成功或非重定向错误
    }
}

// 底层连接逻辑：TCP 三次握手与 TCP_NODELAY
function connect(socket, host, port) {
    return new Promise((resolve, reject) => {
        socket.connect(port, host, () => {
            socket.setNoDelay(true); // 禁用 Nagle 算法，减少延迟
            resolve();
        });
        socket.on('error', reject);
    });
}
```

### 4. 常见误区与进阶思考
误区一：认为‘MOVED’是错误。在 Redis 集群协议中，MOVED 是预期的正常流程指示器，告知客户端其计算的槽位已被迁移至其他节点。专业工程师应将其视为‘重路由指令’而非‘业务异常’，频繁捕获 MOVED 通常意味着客户端未维护最新的 Cluster Slots 元数据缓存，导致网络往返次数激增。

误区二：混淆 PSYNC 与 RDB 恢复时机。许多工程师误以为故障转移时新 Master 会完全重建数据。实际上，Redis 3.0+ 使用 Partial Resynchronization (PSync)，仅在最后一次完整快照（RDB）后产生差异复制缓冲区（Backlog）。若主从断开时间过长或 Backlog 被覆盖，才触发全量同步。理解这点对于评估故障转移期间的数据丢失窗口（Data Loss Window）至关重要。

思考题：在哈希槽正在重平衡（Moving Keys between Nodes）的过程中，如果此时发生故障转移（Failover），Redis 如何保证数据一致性？请结合 ‘Cluster Slots’ 与 ‘Keys’ 的两级映射关系，以及 AOF/RDB 持久化策略进行分析。
