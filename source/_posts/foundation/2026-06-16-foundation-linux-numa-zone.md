---
title: "每日基础技术总结 · 2026-06-16 · Linux 的 NUMA 内存分配策略与 zone 管理"
date: 2026-06-16 08:00:00
categories: [技术分享]
tags: ["技术分享", "操作系统基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-06-16 · Linux 的 NUMA 内存分配策略与 zone 管理

## 📚 今日主题

> **Linux 的 NUMA 内存分配策略与 zone 管理**（操作系统基础）

### 1. 核心概念速览
NUMA（Non-Uniform Memory Access）是一种多处理器系统的内存架构，其本质是内存控制器与处理器集成，导致内存访问延迟随物理距离而不同：访问本地内存（所属node）快，访问远程内存（其他node）慢。Linux通过NUMA调度和内存分配策略，在保持系统全局一致性的前提下，最大化利用局部性，降低平均内存访问延迟。Zone管理是Linux物理内存的分区机制，将每个node的内存划分为DMA、DMA32、Normal、Movable等区，用于应对不同设备的DMA能力限制、隔离可移动页面以支持内存热插拔和防碎片。该机制解决的核心问题是：如何在异构内存拓扑下高效、公平地分配内存，同时满足硬件约束和系统稳定性。在整个计算机体系中，它位于操作系统内存管理核心，与CPU调度、文件系统、容器隔离直接相关。专业工程师必须掌握它，因为性能调优、NUMA感知的进程部署、数据库和虚拟化场景均依赖对内存分配路径和跨节点访问代价的深刻理解。

### 2. 底层原理剖析
Linux内存分配的核心路径是：分配请求经由`alloc_pages`，从当前CPU所在node的zonelist出发，按zone优先级（Normal→DMA32→DMA）遍历，检查zone水位（watermark），若满足则从伙伴系统（buddy allocator）摘取页面；若本地node全部失败，则通过`__alloc_pages_slowpath`进行回收、远程node分配或OOM。NUMA策略嵌入在zonelist的构建中：每个node的zonelist默认按“本地优先”排序（node distance从小到大），且允许通过`numa_zonelist_order`（default/Node/Zone）调整顺序，形成node-local、node-lazy等模式。Zone管理体现在每个zone维护独立的`free_area`和`lowmem_reserve`，且Movable zone内的页面可被migrate，用于反碎片（compaction）。对比前端已有概念：NUMA的本地内存优先策略类似于HTTP缓存中的强制缓存（优先使用本地缓存，未命中才回源），而远程访问则类似协商缓存需要往返验证；Zone管理类似于V8堆中的新生代/老生代分代——不同区域有不同分配策略和回收条件，且对象可在区域间迁移（类似页面迁移）。与Java接口和TS接口的“同名不同语义”类似，NUMA和Zone在抽象上都包含“分层/分区”思想，但一个针对物理延迟，一个针对对象生命周期，各自解决不同问题。

### 3. 基础代码与实战验证
以下为Linux内核页面分配核心流程的精确伪代码（非实际源码，但反映机制）：

```c
// 分配页面入口，gfp_mask 包含分配标志，order 为 2^order 页数
struct page *alloc_pages(gfp_t gfp, unsigned int order) {
    // 1. 获取当前CPU所在NUMA节点编号（本地优先的起点）
    int preferred_nid = numa_node_id();

    // 2. 根据gfp_mask和首选节点构建zonelist（已按距离排序）
    struct zonelist *zl = node_zonelist(preferred_nid, gfp);

    // 3. 快速路径：遍历zonelist中的每个zone（如Normal→DMA32→DMA）
    for_each_zone_zonelist(zone, z, zl) {
        // 检查该zone的水位（WMARK_MIN），确保有足够空闲页
        if (!zone_watermark_ok(zone, order, min_wmark_pages(zone), preferred_nid)) {
            continue;
        }
        // 4. 从该zone的伙伴系统摘取连续页面（可能使用per-cpu page list）
        page = rmqueue(zone, order);
        if (page) {
            return page;  // 本地/近邻成功
        }
    }

    // 5. 慢速路径：本地内存不足时，尝试回收页面、压缩内存或跨节点分配
    page = __alloc_pages_slowpath(gfp, order, preferred_nid);
    return page;
}
```

关键行注释：`numa_node_id()` 体现CPU与内存的亲和性；`node_zonelist` 决定了NUMA策略（如default=本地优先，Node=按节点距离）；`zone_watermark_ok` 是zone管理水位机制的核心，防止低zone内存被耗尽；`rmqueue` 完成实际物理页的摘取。可用 `numactl --hardware` 查看实际NUMA拓扑和节点距离矩阵，用 `numastat` 观察进程的内存分配分布，从而验证本地分配优先策略。

### 4. 常见误区与进阶思考
误区1：认为NUMA机器上“内存总量足够”就不会出现性能问题。实际上，若进程的内存被分配到远程节点，访问延迟可能增加数倍，导致吞吐量骤降。常见错误是忽略CPU绑定（taskset）与内存绑定（numactl --membind）的配合，导致线程调度到node0但内存分配在node1。误区2：将zone管理单纯视为硬件限制（如DMA zone），而忽略Movable zone对内存碎片化的关键作用。Linux的`/proc/pagetypeinfo`展示了不同zone的页面类型分布，若不可移动页面过多，会导致高order分配失败，即使空闲总量充足。思考题：在NUMA节点0内存耗尽、节点1仍有大量空闲内存时，为什么Linux默认不会自动将节点0的进程内存迁移到节点1，而是可能触发swap或OOM？请从内存分配策略的“本地优先”与“回收/迁移代价”权衡的角度分析，并说明什么场景下应调整`numa_zonelist_order`或使用`numad`守护进程。
