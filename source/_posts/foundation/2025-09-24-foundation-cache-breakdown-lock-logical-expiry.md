---
title: "每日基础技术总结 · 2025-09-24 · 缓存击穿与互斥锁/逻辑过期"
date: 2025-09-24 20:00:00
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2025-09-24 · 缓存击穿与互斥锁/逻辑过期

## 📚 今日主题

> **缓存击穿与互斥锁/逻辑过期**（后端基础）

### 1. 核心概念速览
缓存击穿（Cache Breakdown）指单个热点 Key 在过期瞬间，大量并发请求同时穿透缓存到达数据库，导致后端服务过载。与缓存雪崩（Key 批量失效）不同，击穿针对的是极高频的单一数据项。本质是并发控制与时序一致性问题。解决机制分为两类：1. 互斥锁（Mutex Lock）：通过分布式锁确保同一时刻仅有一个线程重建缓存，其余线程阻塞或重试；2. 逻辑过期（Logical Expiration）：Value 中内嵌过期时间字段，读取时不校验是否过期，若发现过期则由后台异步任务重建缓存，当前请求仍返回旧值。此知识点处于高并发系统设计的核心层，直接决定服务的可用性底线，是理解分布式一致性、锁机制及异步处理模型的基础。

### 2. 底层原理剖析
底层运行机制对比如下：
1. 互斥锁机制：
   - 线程 T1 查询缓存未命中，尝试获取 Redis SETNX 锁。
   - T1 成功获得锁，查询 DB，写回缓存，释放锁。
   - 其他线程 T2..Tn 查询缓存未命中，尝试获取锁失败，进入等待队列或短暂休眠后重试。
   - 本质：串行化关键路径，利用锁的可重入性或分布式原子性保证唯一性。前端类比：类似于 JS 事件循环中的 '宏任务' 排队执行，但此处是分布式环境下的硬同步，涉及网络 RTT 和锁粒度开销。

2. 逻辑过期机制：
   - Cache Value = { Data, ExpireTime }。
   - 读取时：Check(Timestamp.now < ExpireTime) ? Return(Data) : AsyncRebuild(Data) && Return(Data)。
   - 重建过程：启动独立线程/协程去查 DB 并更新 Redis Value，主线程无感继续返回。
   - 本质：用空间换时间，牺牲强一致性换取极高读吞吐。前端类比：类似于 Service Worker 中的 'Stale-while-revalidate' 策略，先给旧资源，后台静默更新，用户无感知延迟。

差异点：互斥锁依赖外部协调（Redis/ZK），有网络开销和死锁风险；逻辑过期依赖代码逻辑嵌入，无额外RPC，但可能产生脏数据读窗口。

### 3. 基础代码与实战验证
```text
// 简化版 Java 伪代码演示逻辑过期模式 (Thread-safe logic)
public Object getData(String key) {
    // 1. 从 Redis/Hazelcast 获取对象
    CacheWrapper wrapper = redis.get(key);
    if (wrapper == null) {
        // 完全缺失，通常配合互斥锁处理，此处略
        return loadFromDBAndSetExpire(key);
    }

    // 2. 核心判断：仅比较逻辑时间戳，而非物理 TTL
    boolean isExpired = System.currentTimeMillis() > wrapper.getExpireTime();
    
    // 3. 如果已过期，触发异步重建，但当前线程立即返回旧值
    if (isExpired) {
        rebuildAsync(key); // 启动独立线程池任务
    }
    
    // 4. 无论是否过期，先返回当前可用数据，保证低延迟
    return wrapper.getData();
}

private void rebuildAsync(final String key) {
    threadPool.execute(() -> {
        try {
            // 避免多个过期线程同时打爆DB，内部可加细粒度锁
            Object newData = db.query(key);
            long newExpireTime = System.currentTimeMillis() + TTL;
            // CAS 操作确保最新写入者胜出，防止覆盖未完成的重建
            redis.setexWithTime(key, newData, newExpireTime);
        } catch (Exception e) {
            log.error("Rebuild failed", e);
        }
    });
}
```

### 4. 常见误区与进阶思考
1. 误以为互斥锁能解决所有场景：互斥锁在高并发下会成为新的瓶颈（串行化读写），且存在 Redis 宕机导致的锁丢失或持有超时引发的雪崩风险。逻辑过期虽快，但会导致短暂的数据不一致，需评估业务对实时性的容忍度。
2. 混淆缓存穿透与击穿：穿透是 Key 根本不存在，应使用布隆过滤器或空值缓存；击穿是 Key 存在但刚过期，必须处理并发重建问题。

思考题：在逻辑过期模式下，如果两个不同的线程几乎同时检测到过期并触发 `rebuildAsync`，如何设计机制确保只有一个线程真正查询数据库并更新缓存，而另一个线程放弃重建，同时还能保证后续请求获取到最新数据？提示：考虑乐观锁或版本号机制。
