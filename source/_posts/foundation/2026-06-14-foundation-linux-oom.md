---
title: "每日基础技术总结 · 2026-06-14 · Linux 的 OOM 评分与内存回收策略"
date: 2026-06-14 08:00:00
categories: [技术分享]
tags: ["技术分享", "操作系统基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-06-14 · Linux 的 OOM 评分与内存回收策略

## 📚 今日主题

> **Linux 的 OOM 评分与内存回收策略**（操作系统基础）

### 1. 核心概念速览
Linux OOM（Out Of Memory）评分是内核在物理内存耗尽、且通过常规回收无法获得足够空闲页时，对每个进程计算一个“坏度”分数，并杀死分数最高的进程以释放内存的机制。内存回收策略是内核在内存压力下优先回收页缓存（page cache）、可换出匿名页（swap）等一系列操作。本质：在虚拟内存超售（overcommit）前提下，内核必须通过牺牲进程来恢复内存分配能力的最后手段。解决什么问题：避免系统因内存耗尽而整体崩溃，保证关键进程（如 init）能继续运行。机制：内存不足时，内核首先触发异步/同步回收，通过 LRU 列表回收文件页和匿名页；当回收仍无法满足分配时，唤醒 oom_killer，遍历所有进程，用 oom_badness() 计算分数，选择得分最高的进程发送 SIGKILL。它在操作系统中的地位：属于内核内存管理子系统的核心应急路径，直接影响系统稳定性、容器隔离（cgroup OOM）以及大数据/ AI 训练中的内存管理。专业工程师必须掌握：因为高并发服务、容器编排、机器学习训练经常面临内存压力，理解 OOM 评分和回收策略可以定位“进程被杀”的原因，并能通过 cgroup、oom_score_adj 等工具主动控制风险。

### 2. 底层原理剖析
底层原理：内存回收路径。当内存分配失败（例如在 page fault 处理中），内核调用 __alloc_pages_nodemask，如果空闲页不足，则进入慢路径：唤醒 kswapd 异步回收，若仍不足，则进行直接回收（direct reclaim）。回收的主要对象是页缓存（干净页直接释放，脏页写回）和匿名页（通过 swap 换出）。内核维护 LRU 链表（active/inactive），回收时从 inactive 链表尾部取页。如果 swap 空间不足或匿名页无法换出（如无 swap），则回收效果有限。当所有可回收页被尝试后，仍然没有足够内存，内核调用 out_of_memory() 函数，该函数会遍历进程（排除内核线程、不可杀进程），使用 oom_badness() 计算分数。分数计算公式（现代内核）大致为：`points = (rss + swap + page_table_pages) * 1000 / totalpages`，其中 rss 是常驻内存页数，swap 是进程在 swap 中占用的页数，page_table_pages 是页表占用的页数。然后根据每个进程的 oom_score_adj（范围 -1000 到 1000）调整：`adjusted_points = points + oom_score_adj`，注意这里的 adjusted_points 就是 /proc/pid/oom_score 的值。分数最高者被选中，发送 SIGKILL。特别地，oom_score_adj 设置为 -1000 的进程被视为 OOM_DISABLE，不会参与评分（即不可杀）。与前端已有概念的对比：例如，前端中 JavaScript 引擎（V8）的堆内存管理是进程内回收机制，当堆内存超过限制时触发 GC，若 GC 后仍不足则抛出 OOM 错误——这是语言运行时内的“内存压力应对”，但不会杀其他进程。Linux OOM 则是跨进程的系统级决策，具有强制性和全局性。又如，浏览器对后台标签页的内存回收类似 OOM 评分：基于内存占用和用户可见性选择释放哪个标签页，但这是应用层策略，而 Linux OOM 是内核层。异同：两者都是“资源不足时选择牺牲对象”，但前者基于运行时的对象可达性，后者基于进程的内存权重和系统优先级。

### 3. 基础代码与实战验证
```text
下面给出 C 程序，演示如何读取和调整 oom_score，以及触发内存分配。运行该程序后，通过 dmesg 可观察 OOM killer 行为。注意：该程序本身可能被 OOM killer 杀死，所以请在空闲系统中运行，且确保有足够 swap 或限制内存。代码：

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>

// 读取 /proc/self/oom_score 的值
int read_oom_score(void) {
    char buf[32];
    int fd = open("/proc/self/oom_score", O_RDONLY);
    if (fd < 0) return -1;
    ssize_t n = read(fd, buf, sizeof(buf)-1);
    close(fd);
    if (n <= 0) return -1;
    buf[n] = '\0';
    return atoi(buf);
}

int main(void) {
    printf("PID: %d\n", getpid());
    printf("Initial oom_score: %d\n", read_oom_score());

    // 将 oom_score_adj 调整为 500，使该进程的最终 oom_score 增加 500，
    // 从而在 OOM 时被选中的概率更高。
    int fd = open("/proc/self/oom_score_adj", O_WRONLY);
    if (fd >= 0) {
        write(fd, "500", 3);
        close(fd);
    }
    printf("After oom_score_adj=500, oom_score: %d\n", read_oom_score());

    // 分配 1GB 虚拟内存并逐页写入，触发物理页分配
    size_t size = 1UL * 1024 * 1024 * 1024;
    char *p = malloc(size);
    if (!p) {
        perror("malloc");
        return 1;
    }
    // 每页写入，强制内核分配物理页，否则 malloc 可能只是保留虚拟地址
    for (size_t i = 0; i < size; i += 4096) {
        p[i] = 1;
    }
    printf("Touched %zu bytes, entering pause. Check dmesg for OOM events.\n", size);
    pause();
    return 0;
}

注释已解释底层运作：/proc/self/oom_score 是内核根据 oom_badness 计算出的当前分数；oom_score_adj 直接参与分数线性叠加；memset 或循环写内存会触发缺页异常，从而为进程分配物理页，导致 RSS 上升，使 oom_score 变大。
```

### 4. 常见误区与进阶思考
常见误区 1：认为 oom_score 可以手动设置，或者 oom_score 越小越安全。实际上 oom_score 是内核动态计算的结果，不可直接写；能调整的是 oom_score_adj（或 oom_adj，已弃用）。并且 oom_score 是“坏度”分数，越大越容易被杀，调整 oom_score_adj 为负数可以降低被选中的概率，但无法完全保证（除非 -1000）。误区 2：认为增加 swap 就能完全避免 OOM。swap 只是把匿名页换出，延迟内存耗尽；当 swap 也耗尽，或进程不产生可换出页时，依然会触发 OOM。另外，频繁换出会导致系统 thrashing，性能急剧下降。误区 3：认为 OOM killer 是随机选择进程。实际上基于 badness 评分，评分函数综合考虑进程内存占用和 oom_score_adj，但某些特殊进程（如内核线程、OOM_DISABLE）会被跳过。进阶思考：如果系统中有两个进程 A 和 B，A 的 oom_score_adj 为 -1000，B 的 oom_score 为 50；当内存耗尽时，内核会怎样选择？请解释 oom_badness 中 oom_unkillable 的判断逻辑。这测试你是否理解 OOM_DISABLE 与评分的边界。
