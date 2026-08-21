---
title: "每日基础技术总结 · 2026-06-03 · 分布式锁：Redis Redlock 与 ZooKeeper"
date: 2026-06-03 08:00:00
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-06-03 · 分布式锁：Redis Redlock 与 ZooKeeper

## 📚 今日主题

> **分布式锁：Redis Redlock 与 ZooKeeper**（后端基础）

### 1. 核心概念速览
分布式锁：在分布式系统中，多个进程/节点需要对共享资源进行互斥访问，分布式锁提供跨节点的互斥保证。其本质是依赖一个具有原子操作能力和一致性模型的共享存储系统。Redlock 是 Redis 作者提出的基于多个独立 Redis 主节点实现容错的锁算法；ZooKeeper 则基于 ZAB 协议，利用临时顺序节点与会话机制提供锁服务。分布式锁位于分布式协调中间件层面，是微服务、分布式数据库、分布式任务调度等场景的基础组件。专业工程师必须掌握其底层一致性模型与故障场景，因为选错实现会导致数据竞争或死锁。

### 2. 底层原理剖析
Redlock 原理：客户端获取当前时间 T0，向 N 个（通常为奇数）独立的 Redis 主节点执行 SET key lock_id NX PX ttl，NX 保证键不存在时才设置，PX 设置过期时间。当成功设置的节点数超过 N/2，且整个耗时小于 ttl 时，认为获得锁；否则释放所有已设置节点。释放时执行 Lua 脚本：if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end，通过比较 lock_id 确保只有锁持有者能删除。
ZooKeeper 原理：客户端创建临时顺序节点，路径如 /lock/lock_0000000001，然后获取 /lock 下所有子节点，若自己序号最小则获得锁；否则对前一个序号节点注册 watcher，等待其删除后重新竞争。临时节点与会话绑定，会话超时则节点自动删除，从而避免死锁。
核心差异：Redis 采用异步复制和最终一致性，锁的互斥性在节点故障时可能被破坏；ZooKeeper 通过 ZAB 提供线性一致性，但会话过期可能导致锁提前释放。与前端概念对比：前端中两个标签页可利用 localStorage 和 storage 事件实现跨标签页互斥，但那是单机、无网络分区、无时钟漂移的环境；分布式锁必须处理这些分布式系统特有的问题。这与前端工程师理解 Java 接口与 TypeScript 接口的区别类似，需要关注底层实现差异，而非仅仅 API 表面。

### 3. 基础代码与实战验证
```text
以下用伪代码展示 Redlock 获取与释放锁的核心逻辑（基于 Redis 命令与 Lua 脚本）：

获取锁：
now = current_time_ms()
lock_id = generate_uuid()
for node in redis_nodes:
    result = node.SET('app:lock', lock_id, NX=true, PX=30000)  # NX 保证原子创建，PX 设定过期时间，lock_id 用于安全释放
success_count = count(result == true)
elapsed = current_time_ms() - now
if success_count >= (len(redis_nodes) // 2 + 1) and elapsed < 30000:
    lock_acquired = true
else:
    # 释放已获取的锁，使用 Lua 保证检查与删除原子性
    release_script: if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end
    for node in redis_nodes:
        node.eval(release_script, 1, 'app:lock', lock_id)

ZooKeeper 获取锁伪代码：
lock_id = zk.create('/lock/lock_', ephemeral=true, sequence=true)  # 创建临时顺序节点，返回完整路径如 /lock/lock_000000001
while true:
    children = zk.get_children('/lock')  # 获取所有子节点
    my_seq = extract_sequence(lock_id)
    min_seq = min(extract_sequence(child) for child in children)
    if my_seq == min_seq:
        lock_acquired = true
        break
    else:
        prev_child = child_with_max_seq_less_than(my_seq)  # 找到序号前一个节点
        zk.watch(prev_child, delete_event)  # 监听前一个节点删除事件
        wait_for_event()  # 阻塞等待事件，唤醒后重新竞争
# 释放锁：删除节点即可；若会话过期，节点自动删除
```

### 4. 常见误区与进阶思考
误区一：把 Redlock 当作强一致锁。实际上，Redis 的异步复制和主从切换可能导致锁数据丢失，且客户端在获得锁后发生长时间 STW（Stop-The-World）GC，锁过期后另一个客户端可以获取同一把锁，导致互斥失效。误区二：认为 ZooKeeper 锁绝对安全。ZK 的临时节点与会话绑定，当会话因网络分区或心跳超时而过期时，锁会被自动释放，但原客户端可能仍在执行临界区，此时其他客户端也能获取锁，产生数据竞争。思考题：在 Redlock 场景中，假设客户端 A 获得锁后发生 10 秒 GC，而锁 TTL 为 5 秒，客户端 B 在 A 持锁期间成功获取锁并写入了共享资源。A 恢复后继续执行临界区，如何设计一种机制让 A 能够检测到自己已失去锁并避免写入冲突？提示：可考虑使用 fencing token（单调递增的令牌）或引入版本号校验，使得资源端能够拒绝过期客户端。
