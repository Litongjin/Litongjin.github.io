---
title: "每日基础技术总结 · 2026-07-03 · Cgroups v2 的 CPU 带宽控制：cfs_period/cfs_quota 与 throttling"
date: 2026-07-03 08:00:00
categories: [技术分享]
tags: ["技术分享", "DevOps 与云原生"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-07-03 · Cgroups v2 的 CPU 带宽控制：cfs_period/cfs_quota 与 throttling

## 📚 今日主题

> **Cgroups v2 的 CPU 带宽控制：cfs_period/cfs_quota 与 throttling**（DevOps 与云原生）

### 1. 核心概念速览
CFS（Completely Fair Scheduler）带宽控制是 Linux 内核 cgroup v2 CPU 控制器提供的硬性 CPU 配额机制，由 cpu.max 文件中的两个参数定义：cfs_quota_us（配额）与 cfs_period_us（周期）。配额表示单个周期窗口内该 cgroup 中全部任务可消耗的 CPU 时间上限（微秒）；周期表示配额重置的窗口长度（默认 100000us = 100ms）。两者比值 quota/period 即该 cgroup 可用 CPU 核数的硬性上限（如 50000/100000 = 0.5 核）。本质是『窗口化硬预算』：每周期开始时内核注入满额 quota 预算，任务运行按实际执行时间逐纳秒扣减；预算耗尽且窗口未结束时，该 cgroup 内所有可运行任务被强制移出调度器运行队列（throttled），直到下一周期边界重新注资（unthrottle）。它解决的是多租户场景下 CPU 资源的确定性隔离——与 cpu.weight（权重，软性竞争分配，对应 Kubernetes requests）有本质区别：quota 是即使系统完全空闲也强制生效的硬上限。该机制位于内核调度器层（kernel/sched/fair.c），位于容器运行时（runc/containerd）与编排系统（Kubernetes limits.cpu → cpu.max）之下。后端与 AI 工程师必须掌握它：推理服务、数据管道的 CPU limit 与延迟抖动分析全部建立在这一机制之上，不理解 throttling 就无法对容量规划与尾延迟做出正确判断。

### 2. 底层原理剖析
核心数据结构：每个 task group（cgroup）关联一个 cfs_bandwidth 结构，关键字段为 quota（周期预算总额）、period（窗口长度）、runtime（当前剩余预算，受 cfs_bandwidth_lock 保护）、throttled 状态标志，以及一个高精度 hrtimer（独立于内核 HZ/jiffies，精度纳秒级）。cgroup 内所有任务的调度实体共享同一份 runtime 池。

运行机制按以下状态机运转：
1. 扣减：调度器选择该 cgroup 任务运行时，update_curr → account_cfs_rq_runtime 将实际执行增量 delta_exec 从 runtime 中逐纳秒扣减；该过程发生在每次调度 tick、任务切换及入队/出队时，是连续记账而非批量结算。
2. 节流（throttle）：runtime 降至 0 以下的瞬间调用 throttle_cfs_rq：将 cfs_rq 标记为 throttled，并把运行队列上所有可运行任务摘除，挂入独立的 throttled_list。此后这些任务对 pick_next_task 完全不可见——不排队、不参与调度，等同于被内核强制挂起。
3. 注资（unthrottle）：hrtimer 在周期边界触发 do_sched_cfs_period_timer，执行 refill：runtime 重置为 quota 满额，随后 unthrottle_cfs_rq 将任务重新挂回运行队列。
4. 预算不滚动（no banking）：周期内未用完的 runtime 在窗口结束或任务睡眠时清零，绝不结转。因此这是『每周期独立窗口』的硬预算，而非令牌桶式平滑限速。（内核 ≥6.6 新增 cpu.max.burst 文件，可显式开启未用预算的累积与突发消费，属扩展语义。）

伪代码：
    on_task_runs(delta):
        runtime -= delta
        if runtime <= 0:
            throttle_cfs_rq(cfs_rq)   // 摘除全部可运行任务，arm hrtimer 至下一周期边界
    on_period_timer():
        runtime = quota               // 满额注资
        if cfs_rq.throttled:
            unthrottle_cfs_rq(cfs_rq) // 任务重新入队

时序特征（quota=50ms, period=100ms）：
    |←————— 100ms 周期 —————→|
    |██ 50ms 运行 ██|▨▨ 50ms throttle ▨▨|
    |←—— 下周期边界：注资并 unthrottle ——→|

三个必须理解的边界行为：
- 空闲 CPU 下仍 throttle：即使系统其他 CPU 完全空闲，预算耗尽照样挂起。配额是统计隔离的硬上限，不依赖机会性调度。
- 组内共享预算：单一线程耗尽 runtime 会使同 cgroup 内全部线程连带被 throttle，形成组内相互干扰。
- 突发性：任务可在周期开始瞬间突发消费全部 quota（如连续 50ms 占用 CPU），随后完全挂起；观测到的 CPU 利用率是锯齿波而非平滑 50%。

与前端已有概念的层间辨析（类比『Java interface 与 TS interface』同名异质的对比）：表面相似——浏览器渲染的帧预算（16.7ms/帧）、lodash.throttle 与 CFS quota/period 都是『窗口 + 预算』模型。本质差异有三：(1) 抢占性 vs 协作性：JS 事件循环是协作式的，宏任务一旦执行，浏览器无法中途挂起，只能等其让出；CFS 是内核级抢占，预算耗尽时调度器直接摘除线程，线程无法拒绝，也感知不到被挂起的精确时刻。(2) 共享性：前端不存在『一个任务耗尽预算导致同组其他任务被连带挂起』的模型，CFS 的 runtime 是组级共享池。(3) 强制力：前端帧预算超限只是掉帧或 Long Task 警告，代码继续执行；CFS throttle 是硬隔离，系统空闲也生效。同名的『throttle』在不同层级有完全不同的强制力，这正是前端工程师转向后端最容易产生认知偏差之处。

### 3. 基础代码与实战验证
```text
前置条件：内核 ≥5.0、cgroup v2 已挂载；若 cpu 控制器未启用，先执行 echo '+cpu' > /sys/fs/cgroup/cgroup.subtree_control（需 root）。

以下 C 程序零依赖，进程自行创建 cgroup、设置带宽、移入自身，然后忙等以触发 throttling：

#include <fcntl.h>
#include <stdarg.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/stat.h>
#include <unistd.h>

static void wfile(const char *path, const char *fmt, ...) {
    char buf[128];
    va_list ap;
    va_start(ap, fmt);
    vsnprintf(buf, sizeof buf, fmt, ap);
    va_end(ap);
    int fd = open(path, O_WRONLY);
    if (fd < 0) { perror(path); exit(1); }
    if (write(fd, buf, strlen(buf)) < 0) perror(path);
    close(fd);
}

int main(void) {
    /* 创建 cgroup 子目录：cgroup 本质是内核文件系统接口，mkdir 即创建控制组 */
    mkdir("/sys/fs/cgroup/cpu_demo", 0755);

    /* 写入『quota period』：内核解析后更新 cfs_bandwidth 的 quota 与 period 字段 */
    wfile("/sys/fs/cgroup/cpu_demo/cpu.max", "50000 100000");

    /* 将当前进程移入该 cgroup：内核把 task 挂到对应 cfs_rq，与组内其他任务共享同一份 runtime 池 */
    wfile("/sys/fs/cgroup/cpu_demo/cgroup.procs", "%d", getpid());

    /* 纯 CPU 忙等：调度器持续执行 update_curr → account_cfs_rq_runtime 扣减 runtime，
       约 50ms 后 runtime 耗尽，throttle_cfs_rq 将本进程摘出运行队列，直至周期边界注资 */
    for (volatile unsigned long long i = 0;; i++);
    return 0;
}

编译运行（另开终端观测）：
    gcc -O0 -o cpu_demo cpu_demo.c && ./cpu_demo
    watch -n 0.5 'cat /sys/fs/cgroup/cpu_demo/cpu.stat'

预期结果：nr_periods 随时间递增；nr_throttled 从首个周期起持续递增；throttled_usec 约等于 wall-clock 时间的一半，精确验证『50ms 运行 + 50ms 挂起』的周期行为。用 top 观察该进程 %CPU 同样稳定在约 50%（单核口径）。
```

### 4. 常见误区与进阶思考
误区 1：将 quota/period 当作『平滑速率限制器』或『最低性能保障』。quota 是每周期重置的窗口预算，任务可在周期开头突发跑满全部 quota（连续 50ms 占满 CPU），然后被完全 throttle 至下一周期，观测利用率是『突发 → 0 → 突发』的锯齿波。更反直觉的是：即使系统完全空闲，预算耗尽照样被 throttle——quota 是硬隔离上限，不因有空闲 CPU 而放宽。对延迟敏感服务，这意味着一个 0.5 核限制的服务，在预算耗尽窗口内到达的请求最长要等近一个周期（约 100ms）才能被调度，直接表现为尾延迟尖刺。若需平滑，应调整 cpu.weight 或增大 period（或启用 ≥6.6 的 cpu.max.burst），而不是只调 quota。

误区 2：混淆 cgroup v1/v2 接口与 Kubernetes requests/limits 的语义层级。v1 中带宽参数分散在 cpu.cfs_period_us 与 cpu.cfs_quota_us 两个文件；v2 合并为 cpu.max 单文件（格式『MAX PERIOD』，MAX 可为 max 表示不限），且 v1 的 cpu.shares 对应 v2 的 cpu.weight——机制不同，直接迁移配置必然失效。Kubernetes 中 requests.cpu → cpu.weight（软性竞争权重），limits.cpu → cpu.max（硬性上限）；前者决定竞争时的分配比例，后者决定绝对上限。将两者混为一谈，会在资源超卖与延迟分析中得出系统性错误结论。

进阶思考题：
设某 cgroup 配置 quota=50ms/period=100ms，内含两线程：T1 纯 CPU 忙等，T2 每次运行 1ms 后 I/O 阻塞 9ms。若保持比例不变，将 period 改为 10ms、quota 改为 5ms，T2 从 runnable 到实际执行的最大调度延迟如何变化？请从三层推演：(1) hrtimer 周期粒度细化对预算重置时机的影响；(2) 预算窗口变小后，T1 耗尽预算的节奏加快，T2 落在 throttle 窗口内的概率与最长等待时间如何变化；(3) 更短周期带来更高的定时器中断频率，对系统整体开销与 T2 唤醒延迟的副作用。若你能得出『T2 最大等待从约 50ms 量级降至约 5ms 量级，但代价是定时器开销上升、吞吐量略降』，说明你真正理解了窗口化硬预算的本质。
