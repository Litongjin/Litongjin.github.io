---
title: "每日基础技术总结 · 2026-05-18 · Redis 内存淘汰的近似 LRU/LFU 实现"
date: 2026-05-18 20:00:00
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-05-18 · Redis 内存淘汰的近似 LRU/LFU 实现

## 📚 今日主题

> **Redis 内存淘汰的近似 LRU/LFU 实现**（后端基础）

### 1. 核心概念速览
Redis 内存淘汰策略中的近似 LRU/LFU 机制，本质是在 O(1) 时间复杂度与全局精确统计之间寻求算力与延迟的折中。其核心解决的问题是：在海量键值对场景下，避免使用完整队列或哈希表进行全局排序带来的高昂维护成本（O(N) 或 O(log N)）。该机制通过采样一小部分样本（默认 5 个），从中选出最优者保留，其余剔除。它在计算机体系结构中属于‘空间换时间’与‘概率算法’的典型应用，区别于数据库索引的 B+ 树结构。专业工程师必须掌握它，因为它是理解分布式缓存一致性、热点数据防御以及 Redis 内存管理模型的基础，直接决定高并发场景下的缓存命中率与系统稳定性。

### 2. 底层原理剖析
1. 数据结构基础：Redis 字典（dict）的值指向的是 db 对象（redisDb），其中包含一个 dictEntry 数组。每个 dictEntry 关联一个 serverKey，其元数据中包含 lru（用于 LRU/LFU）和 expires 字段。
2. 采样过程（Sample）：当触发淘汰时，并非扫描整个数据库，而是从当前 DB 中随机选取 N 个键（N=5，可通过 maxmemory-samples 配置）。
3. 淘汰决策（Decision）：
   - LFU 模式：比较采样集中各键的频率计数（counter），淘汰计数最低者。
   - LRU 模式：计算采样集中各键的时钟差（delta = global_clock - key->lru），淘汰时钟差最大者（即最久未访问）。
4. 递归/重试机制：如果选出的淘汰键删除失败（如引用计数保护、AOF/RDB 持久化冲突等），则增加采样次数或进入多轮筛选，直至找到可安全删除的键或达到上限。这保证了最终一定能找到符合 LRU/LFU 语义的空闲内存块。
5. 对比前端概念：这与前端虚拟 DOM 的 Diff 算法有相似之处——不追求全量精准比对，而是在局部范围内做出次优但高效的决策。不同于 TypeScript 接口的静态类型检查（确定性），这是运行时基于采样的概率性优化。

### 3. 基础代码与实战验证
```text
// 伪代码描述 Redis zset 近似 LRU 核心逻辑
function evictionPoolEvict(samplesCount, keys, currentClock) {
    // 1. 初始化候选池，存储 (key, score) 对
    let pool = [];
    
    // 2. 采样阶段：随机选取 samplesCount 个键
    for (let i = 0; i < samplesCount; i++) {
        let randomKey = selectRandomKeyFromDB();
        if (!randomKey) continue;
        
        // 获取该键的内部 lru 值（实际是整数时钟戳，非链表指针）
        let keyLru = getServerKeyLru(randomKey);
        
        // 计算得分：对于 LRU，得分为 (globalClock - keyLru)
        // 得分越高，表示越久未被访问，越适合被淘汰
        let score = currentClock - keyLru;
        
        // 仅将可能成为淘汰对象的键加入候选池
        // 通常只保留得分最高的前几个作为‘待淘汰集合’
        if (pool.length < 10) { // 内部有一个固定的小池大小
             insertIntoPool(pool, randomKey, score);
        } else if (score > pool[0].score) { // 如果比池中最小分还大
             replaceMinInPool(pool, randomKey, score);
        }
    }
    
    // 3. 最终决策：从候选池中选出得分最高者执行 del
    if (pool.isEmpty()) return null;
    
    // 注意：这里返回的可能是被移除的元素，取决于具体实现细节
    // Redis 实际上会在多个采样轮次中不断更新这个池
    return removeBestCandidateFromPool(pool); 
}
```

### 4. 常见误区与进阶思考
误区一：认为近似 LRU 就是精确 LRU。近似 LRU 存在‘长尾效应’盲区，极端情况下，一个极少访问的冷数据可能被误认为是热数据而保留（因为从未被采样到），反之亦然。工程上需根据业务冷热分布调整 maxmemory-samples 大小：样本越多越接近精确 LRU，但 CPU 开销越大；样本越少，误差越大。
误区二：混淆 Redis 的 lru 字段与 JavaScript 中的 WeakMap 清理机制。Redis 的 lru 是一个递增的全局时钟计数器（global_clock），每次访问更新为当前值；JS 的 GC 是基于标记-清除或引用计数的自动垃圾回收，两者原理完全不同。Redis 需要开发者主动设置过期时间或使用淘汰策略，而 JS 引擎自动处理。
思考题：如果我们将 maxmemory-samples 设置为 1，Redis 的近似 LRU/LFU 行为会发生什么退化？这对系统的内存控制精度有什么影响？在实际高吞吐写入场景下，这种退化是否可接受？为什么？
