---
title: "每日基础技术总结 · 2026-10-03 · 高并发秒杀：库存扣减与防超卖"
date: 2026-10-03 07:01:54
categories: [技术分享]
tags: ["技术分享", "分布式与架构设计"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-03 · 高并发秒杀：库存扣减与防超卖

## 📚 今日主题

> **高并发秒杀：库存扣减与防超卖**（分布式与架构设计）

### 1. 核心概念速览
库存扣减与防超卖是秒杀系统的核心问题：在极高并发（成千上万QPS）下，多个请求同时竞争有限库存，系统必须保证扣减后的库存永不为负，且每个成功扣减对应一个有效订单。本质是分布式环境下对共享可变状态（库存）的原子性读-改-写操作。解法分为三类：数据库行锁（悲观锁）、乐观锁（CAS）、Redis Lua 原子脚本。它属于分布式系统与高并发架构设计中的经典一致性场景，核心关注点从『不超卖』延伸至『不重卖、不少卖』。专业工程师必须掌握，因为这是理解分布式事务、缓存一致性、最终一致性的最佳入口，也是面试与线上故障的高频战场。

### 2. 底层原理剖析
底层机制的本质是原子性。任何操作序列若不能在并发下保持『检查库存-扣减库存』的整体性，就会超卖。

1. 数据库悲观锁（SELECT ... FOR UPDATE）
   在事务中对库存行加排他锁，其他事务的读取/更新被阻塞，直到当前事务提交。伪代码：
   BEGIN;
   SELECT stock FROM products WHERE id=? FOR UPDATE;
   IF stock > 0 THEN UPDATE products SET stock=stock-1 WHERE id=?;
   COMMIT;
   锁竞争成为瓶颈，QPS受限于数据库锁粒度与连接数，秒杀场景通常不用。

2. 数据库乐观锁（版本号/CAS）
   不加锁，用版本号或条件更新保证不超卖：
   UPDATE products SET stock=stock-1, version=version+1 WHERE id=? AND stock>0 AND version=?;
   影响行数为1则成功，否则重试/失败。这种条件更新本身就是原子CAS（Compare-And-Swap），避免了显式锁的开销。但每次更新会失效大量请求（ABA问题虽不在此场景，但有版本号可防），且仍需一次DB往返。

3. Redis Lua 原子脚本（秒杀标配）
   将库存扣减逻辑封装在服务端Lua脚本中，Redis保证脚本整体原子执行（单线程事件循环）。核心脚本：
   if redis.call('GET', key) > 0 then
       return redis.call('DECR', key)
   else
       return -1
   end
   更严谨的用 hash 存总库存与已售，DECR 后判断是否小于0，小于则回滚。

   对比前端已有知识：这和前端原子操作（例如 JavaScript 中 `i++` 非原子，但 Web Worker 中 SharedArrayBuffer 的 Atomics.add 是原子）是类似的问题。前端在单线程JS里不需要考虑并发，但有跨线程共享内存时必须用 Atomics 保证原子性；后端的库存扣减在分布式多进程下同样需要原子原语。另一个对比：TypeScript 的接口是编译期结构类型约束，而 Java 接口是运行期多态契约——两者的『约束』层面不同；同样，数据库乐观锁和 Redis Lua 都是『并发约束』，但约束的层级（DB vs 缓存）与失效范围不同。

### 3. 基础代码与实战验证
以下为纯 Redis + Lua 的基础验证示例（不依赖框架），展示最简防超卖原子脚本。

```
# Lua 脚本（扣减库存）
local key = KEYS[1]          -- 库存 key，如 product:100:stock
local stock = tonumber(redis.call('GET', key))
if stock and stock > 0 then
    return redis.call('DECR', key)   -- DECR 是原子操作，但结合前面的 GET 必须用 Lua 包起来
else
    return -1
end
```

运行（伪代码）：
REDIS_EVAL(script, 1, 'product:100:stock')

若返回 >0 表示扣减成功，返回 -1 表示已无库存。

注意：上述脚本中 GET 和 DECR 本身各自原子，但组合起来必须依赖 Lua 脚本保证整体原子性。Redis 执行脚本时不会插入其他命令，因此该脚本是并发安全的。

更严格版本（校验超卖并回滚）：
local key = KEYS[1]
local remain = tonumber(redis.call('DECR', key))
if remain < 0 then
    redis.call('INCR', key)   -- 恢复库存，保证不为负
    return -1
else
    return remain
end

验证原理：在同一个 Redis 连接上并发执行此脚本，无论多少客户端同时发起，Redis 单线程执行脚本，DECR 后 remain 不可能同时为 -1 和 0，因此不会超卖。

### 4. 常见误区与进阶思考
误区一：认为 Redis DECR 本身是原子操作，所以直接 DECR 就能防超卖。
   机制说明：DECR 是原子操作，但它只保证『减1』这一过程不被打断，不保证『扣减前先判断库存是否够』。如果直接执行 DECR，库存可能变成负数（例如库存1时，100个并发请求使库存变为-99），这就是超卖。必须将『判断库存 > 0』与『扣减』放在同一个 Lua 脚本中，或使用条件更新（UPDATE ... WHERE stock > 0），才构成完整的原子逻辑。

误区二：以为数据库悲观锁能保证高性能秒杀。
   FOR UPDATE 锁行虽然不超卖，但将所有请求串行化，数据库连接被占满，吞吐量剧降，无法支撑秒杀峰值。秒杀系统通常将库存预热到 Redis，用 Lua 脚本在 Redis 层完成扣减，再异步落库，本质是『先扣减，后异步对账』，牺牲强一致性换取高并发。

思考题：假设库存为1，两个并发请求同时调用 Redis Lua 脚本，脚本先 GET 库存（值为1），然后 DECR。因为 Lua 是原子执行，请问什么情况下会出现超卖？提示：考虑 Redis 集群模式下的 key 分布与重试逻辑。实际上如果两个请求发往同一个 Redis 节点，Lua 原子性保证不会超卖；但若脚本中使用了多个 key，且这些 key 不在同一个 slot（例如分片集群），Redis 会报错或无法保证原子性。更深层的问题：如果秒杀系统在 Redis 扣减成功，但后面异步写入订单失败，如何处理？——这就是分布式事务中的最终一致性补偿。你能设计出保证『库存扣减成功但订单创建失败时，库存自动回退』的机制吗？
