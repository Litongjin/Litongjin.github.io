---
title: "每日基础技术总结 · 2026-09-25 · Redis 哈希表渐进式 rehash 的扩容与缩容"
date: 2026-09-25 07:05:08
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-25 · Redis 哈希表渐进式 rehash 的扩容与缩容

## 📚 今日主题

> **Redis 哈希表渐进式 rehash 的扩容与缩容**（后端基础）

### 1. 核心概念速览
Redis 的哈希表（dict）采用链地址法解决冲突，底层由两个哈希表（ht[0] 与 ht[1]）构成。渐进式 rehash（incremental rehash）是指当哈希表的负载因子（used/size）超过阈值或低于低水位时，不一次性完成所有桶的迁移，而是将 rehash 过程分散到多次增删改查操作中，每次仅迁移一小部分桶，直至全部完成。其本质是：将 O(n) 的扩容/缩容开销平摊到后续的 O(1) 操作上，避免单次 rehash 导致服务长时间阻塞（尤其是 Redis 单线程模型下的停顿）。该机制是哈希表动态调整容量的经典工程实现，位于 Redis 核心数据结构层，是理解 Redis 内存管理、操作延迟稳定性、以及分布式缓存性能特征的地基。专业工程师必须掌握，因为它是数据结构的缩放一致性与系统可用性之间的核心权衡，直接关系到缓存系统的延迟毛刺与内存占用，也是实现任何高性能哈希容器时不可回避的问题。

### 2. 底层原理剖析
Redis 的 dict 结构包含两个 hash table（ht[0] 与 ht[1]），rehashidx 标记当前迁移进度。正常运行时数据只存在 ht[0]，ht[1] 为空。触发扩容条件：1) 未执行 BGSAVE/BGREWRITEAOF 时负载因子 >= 1；2) 正在执行 BGSAVE/BGREWRITEAOF 时负载因子 >= 5（fork 后写时复制降低内存翻倍风险）。触发缩容条件：负载因子 < 0.1。rehash 过程：

1. 根据 ht[0].used 计算新容量：扩容时为第一个 >= used*2 的 2 的幂；缩容时基于当前 used 计算适当大小。
2. 为 ht[1] 分配新数组，rehashidx 置 0。
3. 每次对 dict 的增删改查（dictAdd、dictFind、dictDelete 等）以及后台定时任务（每次循环最多执行 1ms）都会执行一步 rehash：迁移 ht[0][rehashidx] 整条链上所有元素到 ht[1] 的对应桶（用新 hash & ht[1].sizemask 定位），迁移完成后 rehashidx++，并将原桶置空。
4. 当 rehashidx 达到 ht[0].size 时，释放 ht[0]，令 ht[0] = ht[1]，ht[1] 空置，rehashidx 置 -1。

期间语义：新元素一律插入 ht[1]（避免影响正在收缩的 ht[0]）；查找时先从 ht[0] 查，未命中再到 ht[1]。由此保证数据单调迁移与查询一致性。

与前端已有概念对比：TS 的接口是编译期结构约束，无运行时开销；Redis 的 dict 是运行期自组织的数据结构。渐进式 rehash 类似前端把整页重绘拆分为增量渲染，但本质是内存中数组容量变更的调度分配，而非计算优化——它牺牲内存总量换取时间上的平滑性。

### 3. 基础代码与实战验证
```text
以下为 dict.c 核心逻辑的 C 伪代码，反映底层运作：

typedef struct dictEntry { void *key; void *val; struct dictEntry *next; } dictEntry;
typedef struct dictht { dictEntry **table; unsigned long size; unsigned long sizemask; unsigned long used; } dictht;
typedef struct dict { dictht ht[2]; long rehashidx; /* -1 表示未在 rehash */ } dict;

/* 执行 n 步 rehash，每步迁移一个桶（整条链表） */
int dictRehash(dict *d, int n) {
    while (n-- && d->ht[0].used != 0) {
        dictEntry *de, *nextde;
        /* 跳过空桶 */
        while (d->ht[0].table[d->rehashidx] == NULL)
            d->rehashidx++;
        de = d->ht[0].table[d->rehashidx];
        /* 将桶内所有元素按新掩码重新哈希，头插到 ht[1] 对应桶 */
        while (de) {
            uint64_t h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            nextde = de->next;
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;
            d->ht[0].used--;
            d->ht[1].used++;
            de = nextde;
        }
        d->ht[0].table[d->rehashidx] = NULL;
        d->rehashidx++;
    }
    return d->ht[0].used == 0; /* 1 表示 rehash 完成 */
}

/* 每次增删改查前调用，只迁移一个桶，保证单次操作开销可控 */
int _dictRehashStep(dict *d) {
    return dictRehash(d, 1);
}

/* 后台定时任务调用，在限定时间内持续迁移，避免空闲时 rehash 长期停滞 */
int dictRehashMilliseconds(dict *d, int ms) {
    long long start = timeInMilliseconds();
    do {
        dictRehash(d, 100);
    } while ((timeInMilliseconds() - start) < ms);
    return d->ht[0].used == 0;
}

验证手段：在 Redis 中通过 DEBUG HTSTATS key 查看 ht[0]/ht[1] 槽位与元素数，观察 rehash 过程中两表数据量交替；也可用 INFO 里的 Redis 延迟数据确认无长停顿。
```

### 4. 常见误区与进阶思考
常见误区：

1. 误以为 rehash 过程中所有操作会被全部阻塞。实际是渐进式，每次操作只多承担迁移一个桶的成本（通常微秒级），整体延迟平滑；只有单桶链表极长的极端情况才可能产生明显抖动。
2. 忽略 rehash 期间的内存双倍占用。扩容时 ht[1] 为原表 2 倍大小，迁移中 ht[0] 数据尚未释放，瞬时峰值接近原内存的 3 倍。在内存受限环境下容易触发 maxmemory 淘汰或 OOM，必须结合内存模型与淘汰策略综合评估。

深度思考：若某大 key 的哈希表正处于 rehash 中间阶段，此时用户对该 key 执行 DEL，Redis 会如何正确释放跨越两张表的数据？对于 ht[0] 已迁移、ht[1] 未迁移的部分，以及尚未迁移的桶，删除操作会怎样推进 rehashidx？如果涉及 lazy free 异步线程，又会引入什么并发一致性问题？这需要理解 dictScan、dictDelete 与 rehash 的交互细节才能给出严谨答案。
