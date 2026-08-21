---
title: "每日基础技术总结 · 2026-06-03 · Redis 哨兵模式与主从切换"
date: 2026-06-03 08:00:00
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-06-03 · Redis 哨兵模式与主从切换

## 📚 今日主题

> **Redis 哨兵模式与主从切换**（后端基础）

### 1. 核心概念速览
Redis 哨兵模式（Sentinel）是 Redis 高可用方案的核心控制平面，本质是一个独立运行的分布式进程组，用于监控主从架构中的主节点（master）与从节点（replica），在主节点失效时自动执行故障转移（failover），将某个从节点提升为新主节点，并通知客户端更新连接拓扑。它解决的是 Redis 单点故障导致的可用性问题，即当 master 不可写时，系统无法继续提供服务。哨兵本身是去中心化的，多个哨兵节点通过流言协议（gossip）交换状态，并通过投票（quorum）达成一致，避免误判。在计算机体系中，它属于分布式系统中的故障检测与协调服务，类似于 ZooKeeper 在 Kafka 中的作用，但更轻量且专为 Redis 设计。专业工程师必须掌握它，因为它是构建生产级缓存层的基础，直接关系到数据持久性、服务连续性和分布式系统的一致性理解。

### 2. 底层原理剖析
哨兵模式的底层机制可拆解为三个核心环节：监控（Monitoring）、通知（Notification）、自动故障转移（Automatic Failover）。监控基于周期性 PING 命令，哨兵每秒向所有主从节点发送 PING，若在 down-after-milliseconds 时间内未收到有效回复，则主观判定该节点为不可达（sdown）。为避免网络抖动导致误判，多个哨兵通过 `sentinel is-master-down-by-addr` 命令相互确认，当超过 quorum 数量的哨兵判定同一 master 不可达时，该 master 被标记为客观下线（odown）。故障转移的核心是领导者选举，采用 Raft 算法变体：每个哨兵在发现 master 进入 odown 后，会尝试向其他哨兵请求投票，获得多数票的哨兵成为 leader，负责执行 failover。leader 从从节点中挑选新 master，优先选择复制偏移量（replication offset）最大、即数据最完整的从节点，然后发送 `slaveof no one` 使其成为新 master，并让其他从节点 `slaveof new_master` 重新同步。期间客户端通过哨兵的 `get-master-addr-by-name` 命令动态获取当前 master 地址。这一机制与前端工程中的概念对比：它类似于事件总线中的仲裁机制，但更接近浏览器中的 Service Worker 与主线程的心跳检测——但前端是单机环境，而哨兵要处理分布式下的网络分区（脑裂）问题，因此引入了 quorum 和 epoch（配置纪元）来防止旧 leader 误操作，这类似于前端状态管理库（如 Redux）中的 reducer 纯函数保证确定性，但哨兵需要处理网络不确定性。

### 3. 基础代码与实战验证
以下为使用 Python 连接哨兵并自动切换 master 的极简代码，验证哨兵模式下的客户端感知。

```python
from redis.sentinel import Sentinel

# 连接哨兵节点（不是 Redis 主节点），哨兵负责返回当前 master
sentinel = Sentinel([('127.0.0.1', 26379)], socket_timeout=0.1)

# 获取主节点连接（每次调用都会向哨兵查询当前 master 地址，若发生 failover，会返回新 master）
master = sentinel.master_for('mymaster', socket_timeout=0.1)

# 写数据到当前 master（实际写入新 master 后，旧 master 恢复时会被降级为从节点）
master.set('key', 'value')

# 获取从节点连接（用于读扩展，从节点可能延迟，但能验证主从同步）
slave = sentinel.slave_for('mymaster', socket_timeout=0.1)
print(slave.get('key'))  # 通常能读到刚写入的值，但若同步延迟，可能为 None
```

注释解释：`master_for` 内部通过 `sentinel` 的 `get-master-addr-by-name` 命令获取当前 master IP 和端口，然后建立连接。当哨兵执行 failover 后，新 master 地址会变化，客户端再次调用 `master_for` 会拿到新地址，从而无缝切换。真正的 failover 触发条件需要模拟：停止 master 进程，等待 `down-after-milliseconds` 超时，哨兵会选举并提升新 master。

### 4. 常见误区与进阶思考
常见误区一：认为哨兵可以保证数据零丢失。实际上，failover 时若旧 master 未能及时将数据同步到从节点，会丢失这部分数据，因为新 master 是基于最后一个复制偏移量选的，但未同步的写操作已丢失。这是典型的 CAP 权衡，哨兵优先保证可用性，牺牲了强一致性。常见误区二：忽略脑裂场景——当 master 与哨兵网络分区，但 master 仍在运行，客户端可能继续写入旧 master，而哨兵已提升新 master，导致两个主节点并存。哨兵通过 `down-after-milliseconds` 和 quorum 减少概率，但无法完全避免。正确设计应使用 `min-replicas-to-write` 限制旧 master 写入条件。思考题：若哨兵集群中只有两个哨兵节点，且 quorum 设置为 1，当其中一个哨兵与 master 同侧、另一个在异侧，网络分区时会发生什么？请分析这是否会导致双主，以及如何通过调整配置避免。
