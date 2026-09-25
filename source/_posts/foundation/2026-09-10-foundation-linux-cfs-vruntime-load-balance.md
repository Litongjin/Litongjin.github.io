---
title: "每日基础技术总结 · 2026-09-10 · Linux CFS (Completely Fair Scheduler) 调度器的 vruntime 计算与负载均衡机制"
date: 2026-09-10 08:00:00
categories: [技术分享]
tags: ["技术分享", "操作系统基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-10 · Linux CFS (Completely Fair Scheduler) 调度器的 vruntime 计算与负载均衡机制

## 📚 今日主题

> **Linux CFS (Completely Fair Scheduler) 调度器的 vruntime 计算与负载均衡机制**（操作系统基础）

### 1. 核心概念速览
CFS (Completely Fair Scheduler) 是 Linux 内核默认的 CPU 调度器，核心目标是让所有可运行线程在理想情况下获得等比例于其权重的 CPU 时间份额。它通过维护每个调度实体（sched_entity）的虚拟运行时间（vruntime）来抽象实际运行消耗，并以 vruntime 为键在红黑树中选择下一个待运行任务。该机制取代了传统的时间片轮转，本质上是基于加权公平队列（WFQ）的虚拟时钟调度。它解决的是多任务环境下公平性、响应延迟与吞吐量之间的核心矛盾。在整个计算机/AI 体系中，CFS 位于操作系统进程管理的最底层，直接决定了并发程序的调度延迟与 CPU 资源分配；特别是对于 AI 推理/训练中的多线程算子、容器化 CPU 限制（cfs_quota/cfs_period）、以及微服务高并发时的尾部延迟，都必须理解其内部机制。专业工程师掌握它，才能在系统调优、内核排查、负载均衡设计时做出精准判断而不是靠经验猜测。

### 2. 底层原理剖析
CFS 将 CPU 时间分配建模为虚拟时钟推进问题。每个调度实体有一个 vruntime 字段，其更新公式为：
    vruntime += delta_exec * NICE_0_LOAD / se->load.weight
其中 delta_exec 是实际运行时间，NICE_0_LOAD 是 nice 0 对应的权重（通常为 1024），se->load.weight 由进程的 nice 值映射到全局权重表 sched_prio_to_weight。该公式的本质：权重越大的进程，其 vruntime 增长速度越慢，因此会获得更多实际 CPU 时间。调度器维护一个以 vruntime 为键的红黑树，最左节点（vruntime 最小）即为下一个被选中执行的进程。为了公平启动，新 fork 的进程会被赋予当前 cfs_rq->min_vruntime，防止新进程因攒满 CPU 时间而长期抢占。

每个 CPU 有一个独立的运行队列（cfs_rq），并维护自身的 min_vruntime 和 min_vruntime_copy。CFS 的负载均衡由 scheduler_domain 和 idle_balance 机制驱动：系统周期性触发 load_balance，通过 per-entity load tracking 统计每个运行队列的负载，将任务从负载高的 CPU 迁移到负载低的 CPU。在跨 CPU 迁移时，必须处理 vruntime 的归一化问题，因为不同 cfs_rq 的 min_vruntime 不同。内核通过 entity 的 vruntime 与目标队列 cfs_rq->min_vruntime 之间的差值修正迁移任务的 vruntime（如 place_entity），以确保全局公平性。

与前端知识体系对比：可以类比 React 的调度模型—— React 通过任务优先级和时间切片来控制渲染任务的执行顺序，而 CFS 通过权重和虚拟时钟控制线程的 CPU 占用；但本质区别是 React 的调度是协作式、运行在单线程事件循环上，而 CFS 是内核抢占式，每个线程的 CPU 占用由内核强制执行。另外，CFS 的红黑树结构类似前端性能优化中常用的优先级队列（如最小堆），但红黑树能更好地支持动态调整 key。

### 3. 基础代码与实战验证
```text
以下为 CFS vruntime 核心逻辑的极简伪代码（基于 Linux 内核实际结构抽象），可直接作为理解框架：

/* 调度实体 */
struct sched_entity {
    u64              vruntime;      // 虚拟运行时间，越小越优先
    unsigned long    weight;        // 进程权重，由 nice 值映射得出
    struct rb_node   run_node;      // 红黑树节点，以 vruntime 为 key
};

/* 更新当前进程的 vruntime，在时钟 tick 或调度点调用 */
static void update_curr(struct cfs_rq *cfs_rq) {
    struct sched_entity *curr = cfs_rq->curr;
    u64 now = rq_clock_task(rq_of(cfs_rq));  // 获取当前 CPU 的单调时钟
    u64 delta_exec = now - curr->exec_start; // 本次实际运行时间（纳秒）
    curr->exec_start = now;
    /* 关键公式：虚拟时间 = 实际时间 * 基准权重 / 当前权重 */
    curr->vruntime += delta_exec * NICE_0_LOAD / se->load.weight;
    /* 更新 cfs_rq->min_vruntime，维护运行队列最小虚拟时间 */
    update_min_vruntime(cfs_rq);
}

/* 选择下一个要运行的任务：红黑树最左节点 */
static struct sched_entity *__pick_first_entity(struct cfs_rq *cfs_rq) {
    return rb_entry(rb_first(&cfs_rq->tasks_timeline), struct sched_entity, run_node);
}

/* 新进程加入队列时，避免抢占惩罚：将 vruntime 设置为 min_vruntime */
static void place_entity(struct cfs_rq *cfs_rq, struct sched_entity *se) {
    se->vruntime = max(cfs_rq->min_vruntime, se->vruntime);
}


负载均衡经典步骤伪代码：
1. 计算每个 CPU 运行队列的负载 load = sum(se->load.weight)（现代内核使用 PELT 跟踪平均负载）。
2. 选出 load 最大的源 CPU 和 load 最小的目的 CPU。
3. 从源队列中挑选可迁移的任务（考虑亲和性、缓存降温等）。
4. 对迁移任务的 vruntime 做归一化：
   se->vruntime -= source_cfs_rq->min_vruntime;   // 剥离源队列相对基准
   se->vruntime += dest_cfs_rq->min_vruntime;     // 叠加目标队列基准
5. 将任务插入目标队列的红黑树，并唤醒。

验证思路：在真实 Linux 上可通过 /proc/sched_debug 观察各 CPU 的 vruntime 和负载值，运行两个 nice 不同的 CPU 密集型进程对比实际 CPU 时间分配，即可验证权重公式。
```

### 4. 常见误区与进阶思考
误区 1：认为 nice 值直接决定 CPU 时间百分比。实际上 nice 值只决定权重（weight），CPU 时间份额 = se->weight / 所有可运行进程 weight 之和。例如 nice 0 权重 1024，nice 10 权重 227，则 nice 0 进程获得的 CPU 时间是 nice 10 的约 4.5 倍，但两者绝对比例取决于系统中所有进程的权重总和。另外，CFS 在低频 tick 下存在粒度误差，min_granularity 也会限制最小切换间隔，因此短时间内的分布未必精确。

误区 2：认为同一时刻全局 vruntime 最小的进程必然被调度。实际上每个 CPU 维护独立的 cfs_rq 和 min_vruntime，存在 per-CPU 偏移；跨 CPU 迁移时若不做归一化会破坏公平。并且新唤醒进程、wakeup preempt、带 NUMA 迁移等因素都会通过 place_entity、check_preempt_wakeup 修改 vruntime 或调度决策，红黑树左节点只是基础逻辑，不是最终答案。

深思考题：假设系统有 2 个 CPU，CPU0 上运行一个 nice 0 的进程 A，CPU1 上运行一个 nice 10 的进程 B。若负载均衡将 B 从 CPU1 迁移到 CPU0，如何调整 B 的 vruntime 才能保证迁移后 A 与 B 的全局公平性？如果两个 CPU 的 min_vruntime 相差 1000ms，直接迁移 B 会导致什么现象？请结合 place_entity 和 CFS 的 per-CPU 虚拟时钟独立性来推导正确的归一化公式。
