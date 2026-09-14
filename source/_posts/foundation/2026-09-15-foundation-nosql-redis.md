---
title: "每日基础技术总结 · 2026-09-15 · NoSQL 与 Redis 基础"
date: 2026-09-15 07:03:10
categories: [技术分享]
tags: ["技术分享", "后端基础（Node.js / Java / Python）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-15 · NoSQL 与 Redis 基础

## 📚 今日主题

> **NoSQL 与 Redis 基础**（后端基础（Node.js / Java / Python））

### 1. 核心概念速览
NoSQL：Not Only SQL，指一类非关系型数据管理系统。本质是放弃或弱化关系代数、固定 schema、跨行/跨表 ACID 与 JOIN，以数据模型和访问路径为中心选择存储引擎，换取水平扩展、灵活 schema、高吞吐低延迟。分类：KV（Redis、Memcached）、文档（MongoDB）、列族（HBase、Cassandra）、图（Neo4j）。解决 RDBMS 在超大规模、半结构化、写密集、地理分布、缓存/会话等场景的瓶颈。schema 从数据库转移到应用层；一致性从 ACID 转向 BASE/CAP 权衡。Redis：Remote Dictionary Server，基于内存的 KV 数据结构服务器。数据驻留内存，RESP 协议，单线程命令执行，I/O 多路复用，支持 string/hash/list/set/zset/stream/bitmap/HyperLogLog/GEO。持久化可选 RDB/AOF，复制/哨兵/Cluster。位置：应用与持久层之间的内存数据层，用于缓存、分布式锁、计数器、排行榜、消息流、限流。必须掌握：它是后端 SLA 的核心组件，性能、并发原子性、持久化、复制与一致性决策都依赖它。

### 2. 底层原理剖析
NoSQL 存储层：hash 表 O(1) 点查；B+树范围查询；LSM tree 写优化，顺序写加合并；图邻接表。CAP：网络分区下在一致性和可用性间取舍；BASE：基本可用、软状态、最终一致。
Redis 执行机制：
1. 事件循环：epoll/kqueue/select 监听 listen fd 与连接 fd。
2. 可读事件：read 系统调用读取字节流，RESP 解析器增量解析为命令数组 [cmd, arg...]。
3. 命令表查找：redisCommandTable 找到 proc，校验 arity。
4. 执行：单线程内调用命令实现，操作内存数据结构，结果写入 client 输出缓冲区。
5. 可写事件：write 回写 RESP 回复。
单线程串行执行意味着单命令原子，无需锁；但慢命令阻塞所有客户端。网络 I/O 多路复用使一个线程处理多连接，类似 Node 事件循环，但 Node 将 fs/crypto/DNS 委托 libuv 线程池，Redis 持久化 fork 子进程，命令本身不 yield。
与前端对比：
- JS Map/Set 是语言内存对象，受 GC 和进程边界限制；Redis 数据结构在独立服务进程，跨网络共享，命令原子。
- localStorage 同步阻塞主线程；Redis 客户端异步网络，但服务端执行串行。
- Node 事件循环有宏任务/微任务，Redis 无微任务队列，命令不 await；BLPOP 等阻塞命令挂起客户端但事件循环仍处理其他客户端。
数据结构：sds 二进制安全，O(1) len；dict 哈希表，渐进式 rehash，负载因子触发扩容/缩容；ziplist/listpack 连续内存，省指针，适合小对象；quicklist 双向链表加 listpack；intset 整数集合；zset 用 dict(member->score) 加 skiplist(score,member)，O(1) 查 score，O(logN) 范围。
过期与淘汰：expires dict；惰性删除访问时判断；定期删除随机采样。maxmemory 加 maxmemory-policy：noeviction、lru、lfu、random、ttl。
持久化：RDB fork 加 COW，二进制快照；AOF 追加写，appendfsync always/everysec/no；AOF rewrite。复制：主从异步，psync 全量/部分；Sentinel 故障转移；Cluster 16384 slots。
事务：MULTI/EXEC 队列命令，单线程连续执行，无回滚；WATCH 乐观锁；Lua 原子。

### 3. 基础代码与实战验证
```text
# 以下命令通过 redis-cli 发送 RESP 到 127.0.0.1:6379，验证单线程串行与数据结构。
redis-cli SET k v
# SET 命令在 dict 中写入 key=k、value=v；单线程内完成，返回 +OK。
redis-cli GET k
# GET 读取同一 dict；命令边界之间不会插入其他客户端命令。
redis-cli INCR counter
redis-cli INCR counter
# 两次 INCR 返回 1、2；单线程内“读-加-写”原子，不会交叉。
redis-cli EXPIRE counter 10
redis-cli TTL counter
# EXPIRE 写入 expires dict；TTL 读取剩余时间；过期由惰性删除和定期采样清理。
redis-cli ZADD rank 100 alice 90 bob
redis-cli ZREVRANGE rank 0 -1 WITHSCORES
# ZADD 同时更新 dict 与 skiplist；ZREVRANGE 从 skiplist 反向扫描，O(logN+M)。
redis-cli EVAL 'return redis.call(ARGV[1], KEYS[1])' 1 k GET
# EVAL 在单线程内执行 Lua，脚本内命令作为一个原子整体，不与其它客户端命令交错。
# 关键结论：单条命令或单个 Lua 脚本原子；跨多条独立命令不原子，需 WATCH/MULTI 或 Lua 保证读改写。
```

### 4. 常见误区与进阶思考
误区1：Redis 单线程等于无并发问题或不会阻塞。纠正：单线程指命令执行串行；连接、网络、持久化 fork、AOF fsync 等仍可并发/阻塞。慢命令 O(N) KEYS、大集合、长 Lua、bigkey 删除会阻塞整个实例；客户端并发只是命令排队。
误区2：NoSQL 无 schema 且最终一致，所以可随意存。纠正：schema 转移到应用层，索引、迁移、校验需自行实现；Redis 事务有原子性无回滚；AOF everysec 可能丢 1 秒数据；主从异步复制切换可能丢写；缓存与 DB 双写一致性需业务补偿。
思考题：两个客户端同时执行 GET k=1，然后各自 INCR k，最终是 2 还是 3？为什么在 Redis 单线程模型下仍可能丢失更新？给出用 WATCH/MULTI/EXEC 或 Lua 消除的代码级思路，并说明网络重试、超时、主从切换下分布式锁需要额外考虑什么（fencing token、续期、时钟漂移、GC/STW）。这检验命令边界与原子性范围。
