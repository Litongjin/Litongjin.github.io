---
title: "每日基础技术总结 · 2026-08-06 · Redis 的过期删除策略：惰性删除与定期删除及近似 LRU 淘汰"
date: 2026-08-06 08:00:00
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-08-06 · Redis 的过期删除策略：惰性删除与定期删除及近似 LRU 淘汰

## 📚 今日主题

> **Redis 的过期删除策略：惰性删除与定期删除及近似 LRU 淘汰**（后端基础）

### 1. 核心概念速览
Redis的过期删除策略是内存数据库管理已设置TTL的key生命周期的核心机制。惰性删除（lazy expiration）指在每次读取key时检查其是否超过过期时间，若过期则立即删除并返回空；定期删除（active expiration）指Redis的后台定时任务（serverCron）周期性地从过期字典中随机抽样检查并删除过期key，以弥补惰性删除对无访问key的遗漏。近似LRU（approximate LRU）则是在Redis达到maxmemory上限时触发的内存淘汰策略，它并不直接删除过期key，而是通过采样近似地淘汰最久未使用的key以释放内存。该机制解决了有限内存下缓存命中率与CPU开销之间的权衡问题。在计算机体系中，它与操作系统页面置换、CPU缓存替换策略同源，是存储引擎的核心设计。专业工程师必须掌握，因为正确配置maxmemory-policy、过期时间与采样参数，是避免缓存雪崩、穿透，保证Redis高可用和性能的关键。

### 2. 底层原理剖析
惰性删除的底层实现：每个数据库维护一个expires字典（过期字典），键为指向key的指针，值为过期时间戳（毫秒）。在执行任何读写命令时，Redis调用expireIfNeeded()函数，伪代码如下：

    def expireIfNeeded(key):
        if key not in expires: return False
        if current_time_ms >= expires[key]:
            deleteKey(key)
            return True
        return False

该函数在命令真正执行前被调用，因此过期key不会被返回。但未被访问的过期key会一直占用内存。

定期删除的底层实现：Redis的主循环以每秒10次的频率调用activeExpireCycle()。它遍历所有数据库，从expires字典中随机抽取ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP个（默认20）key，检查是否过期并删除。如果过期的比例超过25%，则继续抽取；整个循环受时间预算限制（例如1ms），超出即退出，避免阻塞事件循环。伪代码：

    def activeExpireCycle():
        for db in databases:
            do:
                sample = random_sample(db.expires, 20)
                expired = [k for k in sample if is_expired(k)]
                delete_all(expired)
                if len(expired) / len(sample) < 0.25: break
            while time_elapsed < 1ms

近似LRU的底层实现：当内存超过maxmemory且策略为allkeys-lru或volatile-lru时，Redis在每次命令处理前尝试内存回收。它并不维护全局LRU链表，而是每个key维护一个24bit的LRU时钟（即上次访问时间，秒级精度）。淘汰时随机抽取N个key（默认N=5，由maxmemory-samples配置），选择LRU时钟最小的key淘汰。此外，Redis维护一个淘汰候选池，在多次抽样中将更优的候选放入池中，从而提升淘汰质量。伪代码：

    def evict():
        pool = []
        while memory_usage > maxmemory:
            sample = random_sample(all_keys, 5)
            victim = min(sample, key=lambda k: k.lru)
            pool.append(victim)
            if len(pool) >= 10: pool.sort_by_lru(); pool = pool[:5]
            victim = pool[0]
            delete(victim)

与前端已有概念的对比：惰性删除类似前端在读取localStorage时手动检查时间戳，都是访问时验证；定期删除类似前端用setInterval定时清理本地缓存；近似LRU类似前端实现LRU缓存（如使用Map+双向链表），但前端通常可以维护精确的访问顺序，而Redis受单线程和内存开销限制，采用采样+时间戳近似。核心区别在于Redis是单线程事件循环，任何操作都不能长时间阻塞，因此需要时间预算和随机采样来平衡。

### 3. 基础代码与实战验证
```text
以下Python代码模拟了Redis的惰性删除、定期删除和近似LRU淘汰的底层逻辑。

import time, random

class RedisMock:
    def __init__(self, maxmemory=100):
        self.maxmemory = maxmemory
        self.store = {}  # key -> [value, expire_at, last_access]

    def set(self, key, value, ttl=None):
        expire_at = time.time() + ttl if ttl else None
        self.store[key] = [value, expire_at, time.time()]  # 记录最近访问时间
        if len(self.store) > self.maxmemory:
            self._evict()  # 超过内存上限，触发近似LRU淘汰

    def get(self, key):
        item = self.store.get(key)
        if not item:
            return None
        # 惰性删除：访问时检查过期时间
        if item[1] is not None and time.time() > item[1]:
            del self.store[key]  # 过期即删除
            return None
        item[2] = time.time()  # 更新最近访问时间
        return item[0]

    def active_expire_cycle(self, sample_size=20):
        # 定期删除：随机抽样，避免全量扫描阻塞
        keys = random.sample(list(self.store.keys()), min(sample_size, len(self.store)))
        expired = 0
        now = time.time()
        for key in keys:
            item = self.store.get(key)
            if item and item[1] is not None and now > item[1]:
                del self.store[key]
                expired += 1
        # 过期比例超过25%则继续抽样（受时间预算限制，此处简化）
        if keys and expired / len(keys) > 0.25:
            self.active_expire_cycle(sample_size)

    def _evict(self):
        # 近似LRU：随机采样5个key，淘汰最久未访问的
        sample = random.sample(list(self.store.keys()), min(5, len(self.store)))
        victim = min(sample, key=lambda k: self.store[k][2])  # 按last_access排序
        del self.store[victim]

关键点：get方法体现了惰性删除；active_expire_cycle体现定期删除的概率抽样与比例反馈；_evict体现近似LRU的采样与最小LRU淘汰。
```

### 4. 常见误区与进阶思考
常见误区1：混淆“过期删除”与“内存淘汰”。过期删除只删除已过期的key，是TTL语义的保证；近似LRU是内存压力下的主动淘汰，与TTL无关，可能删除尚未过期的key。

常见误区2：认为定期删除会扫描全库。实际是随机抽样，且有CPU时间上限，因此过期key的删除是概率性的，未访问且未被抽样的过期key会长期占用内存。

思考题：若Redis内存远未达到maxmemory，一个TTL为60秒的key在60秒后从未被访问，它能否被永久保留？结合定期删除的抽样逻辑和惰性删除的触发条件，分析其被删除的期望时间与哪些参数相关。
