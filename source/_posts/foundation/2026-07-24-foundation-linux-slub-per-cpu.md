---
title: "每日基础技术总结 · 2026-07-24 · Linux 的 slub 分配器与 per-CPU 页缓存"
date: 2026-07-24 08:00:00
categories: [技术分享]
tags: ["技术分享", "架构与设计"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-24 · Linux 的 slub 分配器与 per-CPU 页缓存

## 📚 今日主题

> **Linux 的 slub 分配器与 per-CPU 页缓存**（架构与设计）

### 1. 核心概念速览
SLUB 是 Linux 内核默认的通用 slab 分配器，用于管理内核中频繁创建和销毁的小对象（通常小于一个页）。其本质是在伙伴系统（Buddy System）之上构建的一层对象缓存，核心机制是复用已初始化的对象，避免重复的页分配与对象构造开销。per-CPU 页缓存（Page Cache）则指每个 CPU 维护的页分配热缓存，用于加速单页内存的分配与释放，减少对全局伙伴系统锁的竞争。二者共同解决的核心问题是：内核内存分配的频率极高，而伙伴系统以页为单位、需持有全局锁，直接使用会导致严重的性能瓶颈和碎片化。SLUB 通过 per-CPU freelist 实现无锁或低锁的对象分配，per-CPU 页缓存则通过每 CPU 预留少量页框实现快速单页分配。该机制位于内核内存管理子系统的核心层，是所有内核对象（如 task_struct、inode、dentry、socket 缓冲区）创建的基石。专业工程师必须掌握它，因为它是理解内存分配性能、内核锁竞争、NUMA 亲和性、内存碎片及 OOM 行为的基础；对于后端服务而言，系统调用、网络包处理、文件操作等均依赖这些底层分配路径，理解其原理有助于诊断性能问题并优化应用行为。

### 2. 底层原理剖析
SLUB 分配器的核心数据结构是 kmem_cache，每个 cache 管理一类固定大小的对象。每个 CPU 维护一个本地 freelist（当前 CPU 的 partial 列表），分配对象时优先从本地 freelist 取，无锁；本地为空则从节点的 partial 列表中批量搬移对象到本地，或向伙伴系统申请新页。释放对象时优先放回本地 freelist，若本地过多则回收到 partial 列表。SLUB 的关键设计是取消传统 slab 中复杂着色和队列，简化元数据，将空闲对象指针直接存储在对象内存本身（object 首字），降低内存开销。其路径可表示为：kmem_cache_alloc → this_cpu_cmpxchg 取对象 → 失败则 __slab_alloc 走 slowpath 从 partial 或新页补充。释放路径类似。per-CPU 页缓存由 struct per_cpu_pages 维护，内含链表数组（按迁移类型和阶数），当分配一个页框时，优先从当前 CPU 的链表头取；若链表为空，则从伙伴系统批量搬运（如 2^batch 个页）补充，减少对 zone->lock 的争用。释放单页时也先放入本地缓存，达到高水位时批量回收到伙伴系统。这一机制与前端概念对比：前端中 '接口' 有两种理解——TypeScript 的 interface 是编译期结构契约，无运行时存在；Java 的 interface 是运行时多态的基元，有虚方法表。而 SLUB 与 per-CPU 页缓存的关系类似于：TS 的 interface 只是声明（伙伴系统的页分配规则），而 Java 的 interface 是实际对象（per-CPU 缓存实例），二者层次不同。更贴切的对比是前端构建系统中的模块缓存：Webpack 的 module cache 类似 per-CPU 页缓存，为热模块快速返回，而 LRU 缓存与 SLUB 的 partial 列表有相似之处——但核心差异在于 SLUB 不仅缓存内存块，还维护对象生命周期（构造/析构），这类似前端对象池（Object Pool）但位于内核态，且无 GC，必须显式释放。

### 3. 基础代码与实战验证
```text
伪代码验证（无框架，描述内核逻辑）：

// 分配一个内核对象（如 struct file）
void *kmem_cache_alloc(struct kmem_cache *s, gfp_t gfp) {
    // 1. 获取当前 CPU 的 freelist 头部
    void *object = this_cpu_read(s->cpu_slab->freelist);

    // 2. 尝试无锁弹出第一个对象（原子操作）
    if (object) {
        // 将 freelist 后移，并更新当前 CPU 的 freelist 指针
        this_cpu_write(s->cpu_slab->freelist, get_freepointer(s, object));
        // 对象即内存地址，直接返回；无需清零（若无构造要求）
        return object;
    }

    // 3. 本地无对象，进入慢速路径：从 partial 列表补充或新页
    object = ___slab_alloc(s, gfp);
    // ___slab_alloc 内部：
    //   a. 从节点 partial 列表获取 slab（含若干空闲对象）
    //   b. 若没有，则从伙伴系统获取新页，构造新的 slab
    //   c. 批量填充当前 CPU 的 freelist
    return object;
}

// 释放一个内核对象
void kmem_cache_free(struct kmem_cache *s, void *x) {
    // 1. 检查对象是否属于本 cache（调试时校验）
    // 2. 将对象头写入 freelist 指针，形成链表节点
    set_freepointer(s, x, this_cpu_read(s->cpu_slab->freelist));
    // 3. 将对象挂到当前 CPU freelist 头部
    this_cpu_write(s->cpu_slab->freelist, x);
    // 4. 若当前 CPU freelist 长度超过阈值，则批量回收到 partial 或伙伴系统
    if (unlikely(this_cpu_read(s->cpu_slab->objects) > s->cpu_partial))
        __slab_free(s, x); // 回收部分到节点 partial
}

// per-CPU 页缓存分配单页（简化）
struct page *rmqueue_pcplist(struct zone *z, int order) {
    if (order == 0) { // 单页请求
        struct per_cpu_pages *pcp = this_cpu_ptr(z->pageset);
        struct list_head *list = &pcp->lists[get_migratetype()];
        if (list_empty(list)) {
            // 批量从伙伴系统搬移 pcp->batch 个页到本地链表
            rmqueue_bulk(z, order, pcp->batch, list);
        }
        // 从链表头部摘下一个页并返回
        page = list_first_entry(list, struct page, lru);
        list_del(&page->lru);
        return page;
    }
    // 高阶分配直接走伙伴系统
}

注释：上述伪代码展示了无锁快速路径（fastpath）和批量补充的慢速路径（slowpath）。前端类似概念是 React 的 Fiber 复用：React 在协调过程中复用旧的 Fiber 节点（对象池），避免重新创建，但其更新队列和优先级调度远比 SLUB 复杂；SLUB 的快速路径本质上是 CAS 操作单链表头部，而 React 的复用需要遍历树，二者在并发模型和延迟要求上不同。
```

### 4. 常见误区与进阶思考
误区1：认为 SLUB 和 per-CPU 页缓存是同一层级的两种独立机制。实际上，SLUB 本身也使用 per-CPU 逻辑，并且当 SLUB 需要新页时，会调用伙伴系统分配，而伙伴系统的单页分配会优先经过 per-CPU 页缓存。因此 per-CPU 页缓存是 SLUB 的基石之一，二者构成层级关系，而非并列。误区2：认为 SLUB 只是简单地缓存空闲对象，类似内存池。实际上 SLUB 的核心在于利用对象本身存储空闲指针，避免了额外的元数据开销，并且通过 per-CPU 局部缓存极大减少锁竞争；同时 SLUB 支持 NUMA 感知和 CPU 迁移（迁移后对象会跨节点访问），忽略这些会导致对性能瓶颈的误判。

思考题：在 SLUB 中，当 CPU 数量很多且分配频率极高时，per-CPU freelist 会显著减少锁竞争，但为什么在多核场景下仍然可能出现 kmem_cache_alloc 的 slowpath 成为瓶颈？请从以下角度分析：当某个 CPU 的本地 freelist 耗尽时，它需要从节点 partial 列表搬移对象，而 partial 列表是节点共享的，需要持有节点锁；如果多个 CPU 同时触发 slowpath，锁争用就会加剧。但更深层的问题是：为什么 partial 列表会经常为空？这可能是因为释放操作也走 per-CPU freelist，导致对象不会及时回到 partial，从而在分配时产生突发性 empty。另一个原因是 SLUB 的 cpu_partial 阈值决定了对象回收到 partial 的时机，若阈值设置不当，会导致 partial 列表长度波动。真正理解 SLUB 需要进一步思考：SLUB 如何通过 per-CPU partial 和 node partial 的同步设计来平衡分配延迟与内存利用率？你能推导出在 NUMA 系统上，当任务频繁迁移时，对象缓存如何影响内存访问局部性？请尝试用 cache miss 率和互连总线流量来解释。
