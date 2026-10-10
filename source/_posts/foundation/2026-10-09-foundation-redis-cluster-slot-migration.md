---
title: "每日基础技术总结 · 2026-10-09 · Redis集群Slot分配算法与Hash Slot移动的一致性保证"
date: 2026-10-09 08:00:00
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-09 · Redis集群Slot分配算法与Hash Slot移动的一致性保证

## 📚 今日主题

> **Redis集群Slot分配算法与Hash Slot移动的一致性保证**（后端基础）

### 1. 核心概念速览
Redis Cluster 将键空间划分为固定 16384 个 hash slot，每个节点负责若干 slot。Slot 分配算法：对 key 执行 CRC16-CCITT（多项式 0x1021）得到 16 位校验值，取低 14 位，即 slot = crc16(key) & 16383；若 key 含 {tag}，则只对 tag 计算，用于将多个 key 强制路由到同一 slot。Hash slot 移动（resharding）解决分布式缓存在线水平扩展与 slot 归属变更问题：在不停止集群服务的前提下，将某个 slot 的所有 key 从源节点迁移至目标节点，并保证迁移期间单 key 读写的正确性，避免丢失、重复或路由错乱。在分布式体系中，它属于数据分区路由与所有权转移的一致性协议层，是 Redis Cluster 区别于普通哈希分片的关键机制；专业工程师必须掌握它，因为客户端路由、多 key 操作限制、在线扩容与自动化运维都依赖其语义。

### 2. 底层原理剖析
Redis 选择 16384 个 slot 是 CRC16 输出 16 位、取低 14 位的结果，也是心跳包中 slot 位图大小与最大节点数之间的权衡。slot→node 映射由所有节点通过 Gossip 协议维护成一致的 cluster state；客户端缓存该映射，并对每个 key 计算 slot 后直接发送到目标节点。

迁移开始前，目标节点执行 CLUSTER SETSLOT <slot> IMPORTING <source>，源节点执行 CLUSTER SETSLOT <slot> MIGRATING <target>；此时 slot 的 owner 仍为源节点。之后对 slot 内每个 key 使用 MIGRATE 命令原子迁移：源节点单线程执行 DUMP 序列化、发送到目标节点 RESTORE，目标节点成功响应后源节点删除该 key；若任意一步失败，源节点保留 key，目标节点无半成品，因此单 key 不会双写或丢失。

迁移期间的请求路由规则：
1. 节点收到不属于自己的 slot 请求时，先检查该 slot 是否处于 IMPORTING 且当前连接已发送 ASKING；是则执行本地查询，否则返回 -MOVED slot owner。
2. 若 slot 属于本节点且处于 MIGRATING：如果 key 仍在本地则直接执行；如果 key 已迁移走则返回 -ASK slot target。
3. 客户端收到 ASK 后，只对本次请求向目标节点发送 ASKING，然后重发原命令；不得更新 slot→node 缓存。ASK 是临时重定向，因为 slot 归属尚未改变。
4. 整个 slot 迁移完成后，源和目标执行 CLUSTER SETSLOT <slot> NODE <target> 完成归属变更，slot→node 映射更新为 target，此后非目标节点返回 MOVED。归属变更还伴随 config epoch 递增，用于在并发配置下确定最新配置。

与前端路由对比：MOVED 类似带缓存更新的永久 301/302 重定向，客户端收到后应写入新的 slot 路由；ASK 类似临时 307 重定向，仅当前请求有效，不污染缓存。不同点在于 Redis 的 slot 归属由集群内部配置 epoch 与 Gossip 状态机保证最终一致，前端路由表则通常由服务端或网关静态下发。

### 3. 基础代码与实战验证
```text
import re

# Redis 使用 CRC16-CCITT（poly 0x1021）对 key 或 hash tag 计算 16 位校验值
def crc16_ccitt(data: bytes) -> int:
    crc = 0x0000
    for b in data:
        crc ^= b << 8
        for _ in range(8):
            if crc & 0x8000:
                crc = ((crc << 1) ^ 0x1021) & 0xFFFF
            else:
                crc = (crc << 1) & 0xFFFF
    return crc

def redis_slot(key: str) -> int:
    # 若 key 中存在非空 {tag}，只对 tag 计算，保证同 tag 的多个 key 落在同一 slot
    if '{' in key and '}' in key:
        tag = key.split('{')[1].split('}')[0]
        if tag:
            key = tag
    return crc16_ccitt(key.encode()) & 16383

# Redis Cluster 官方特性：只取 tag，所以 foo{user1000}followers 与 user1000 同 slot
assert redis_slot('foo{user1000}followers') == redis_slot('user1000')
assert 0 <= redis_slot('any_key') < 16384

class Node:
    def __init__(self, name):
        self.name = name
        self.data = {}
        self.importing = {}   # slot -> source
        self.migrating = {}   # slot -> target

class Cluster:
    def __init__(self):
        self.nodes = {}
        self.slot_owner = [None] * 16384

    def set_owner(self, slot, node):
        self.slot_owner[slot] = node

    def handle(self, node, conn, cmd, key):
        s = redis_slot(key)

        # 关键：即使 slot 的 owner 不是本节点，只要本节点处于 IMPORTING
        # 且连接发过 ASKING，就允许处理请求；这是目标节点接收 ASK 的入口
        if s in node.importing and conn.get('asking'):
            return self._exec(node, cmd, key)

        owner = self.slot_owner[s]
        if node != owner:
            # 永久归属已变更：返回 MOVED，客户端应更新 slot -> node 缓存
            return ('MOVED', s, owner.name)

        if s in node.migrating:
            target = node.migrating[s]
            if key in node.data:
                # key 仍在源节点：直接本地执行，不重定向
                return self._exec(node, cmd, key)
            # key 已被 MIGRATE 迁移到目标节点：返回 ASK，仅临时重定向
            return ('ASK', s, target.name)

        return self._exec(node, cmd, key)

    def _exec(self, node, cmd, key):
        if cmd == 'GET':
            return node.data.get(key)
        if cmd == 'SET':
            node.data[key] = 'val'
            return 'OK'

    def migrate_key(self, src, dst, key):
        s = redis_slot(key)
        # 标记迁移状态：源 MIGRATING、目标 IMPORTING，slot_owner 仍指向源
        src.migrating[s] = dst
        dst.importing[s] = src
        # 真实 Redis 中由 MIGRATE 在源端单线程原子执行 DUMP+RESTORE+DEL
        dst.data[key] = src.data.pop(key)
        # 整个 slot 迁移完成后才更新 slot_owner[s] 并清除两侧迁移状态

# 场景验证
cluster = Cluster()
a = Node('A'); b = Node('B')
cluster.nodes = {'A': a, 'B': b}
s = redis_slot('foo')
cluster.set_owner(s, a)
a.data['foo'] = 'old'
cluster.migrate_key(a, b, 'foo')

print(cluster.handle(a, {}, 'GET', 'foo'))           # ('ASK', s, 'B')
print(cluster.handle(b, {'asking': True}, 'GET', 'foo'))  # 'old'
print(cluster.handle(b, {}, 'GET', 'foo'))           # ('MOVED', s, 'A')
```

### 4. 常见误区与进阶思考
误区一：把 ASK 当成 MOVED 并更新客户端 slot 缓存。ASK 仅是迁移中的临时重定向，目标节点只对当前连接收到 ASKING 后的单次请求放行；若缓存放为 target，后续请求会绕过源节点，迁移失败回滚后会造成路由错误。

误区二：认为 hash slot 迁移期间多 key 操作仍具备原子性。Redis Cluster 只保证单 key MIGRATE 的原子性；MGET/MSET 等命令若涉及迁移中的 slot，可能因为部分 key 已在目标、部分仍在源而返回 TRYAGAIN 或产生非预期结果。在线 resharding 时应避免对迁移 slot 使用多 key 命令。

进阶思考：为什么 Redis 在 slot 迁移期间不采用读锁/写锁阻塞该 slot 的所有请求，而选择 ASK/ASKING 重定向？这种设计在一致性、可用性和客户端复杂度之间做了什么权衡？提示：从迁移耗时、流量放大、客户端路由缓存与单 key 原子性四个维度分析。
