---
title: "每日基础技术总结 · 2026-10-20 · mmap 映射文件的缺页中断处理流程与脏页回写策略"
date: 2026-10-20 08:00:00
categories: [技术分享]
tags: ["技术分享", "操作系统基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-20 · mmap 映射文件的缺页中断处理流程与脏页回写策略

## 📚 今日主题

> **mmap 映射文件的缺页中断处理流程与脏页回写策略**（操作系统基础）

### 1. 核心概念速览
mmap 将磁盘文件或设备直接映射至进程虚拟地址空间（VMA，vm_area_struct），把「文件 I/O」转化为「内存访问」。其核心机制有二：一是 缺页中断（Page Fault），首次访问未建立页表项的虚拟地址时，CPU 触发 #PF 异常，陷入内核态，由 handle_mm_fault 根据 VMA 调用文件系统的 fault 钩子（如 filemap_fault），从页缓存（Page Cache）查找或从磁盘读入数据页，建立 PTE（Page Table Entry）后恢复执行；二是 脏页回写（Writeback），修改后的页被标记为 Dirty（PTE.D 位置位），内核通过 writeback 机制（pdflush/flush 线程）异步或同步（msync/fsync）将脏页刷回底层块设备。它解决了传统 read/write 系统调用存在的 用户态-内核态数据拷贝 与 上下文切换开销 问题，实现零拷贝（Zero-Copy）与按需分页（Demand Paging）。在 AI 体系中，模型权重文件、Embedding 表常通过 mmap 实现惰性加载与内存共享；专业工程师必须掌握它，否则无法精准分析 OOM、脏页堆积导致的 IO 抖动或内存泄漏假象。

### 2. 底层原理剖析
底层机制分两条路径：

【读路径：Page Fault 处理流程】
1. 用户态访问 mmap 地址 → CPU 检查 PTE，若 Present=0 → 触发 #PF，CR2 寄存器记录 fault address。
2. 内核进入 do_page_fault → 查找 VMA，若地址在 mmap 范围内且 VMA 的 vm_ops->fault 非空 → 进入文件映射缺页处理（非匿名页）。
3. 调用 filemap_fault：先查 Page Cache（基数树/ XArray），命中则直接建立 PTE；未命中则调用 readpage 发起块层 I/O 从磁盘读入 4KB 页，加入 Page Cache，再建立 PTE（Read-Only 或 Read-Write 视 VMA 权限）。
4. 返回用户态，重新执行触发指令，此时指令正常执行。

【写路径：脏页标记与回写策略】
1. 写访问时，若 PTE 为 Read-Only（COW 或 MAP_PRIVATE 触发）或 Dirty 位需更新，CPU 置位 PTE.D（Dirty），页被标记为脏页。
2. 内核通过「周期回写」与「阈值回写」控制：
   - 周期回写：flusher 线程每 dirty_writeback_centisecs（默认 500，即 5s）唤醒，扫描超过 dirty_expire_centisecs（默认 3000，即 30s）的脏页。
   - 阈值回写：当脏页占系统总内存比例超过 dirty_background_ratio（默认 10%）时，flusher 开始异步回写；若超过 dirty_ratio（默认 20%），触发同步回写，写进程被阻塞在 balance_dirty_pages，防止脏页无限堆积。
3. 回写时，clear_page_dirty_for_io 清除 Dirty 位，调用 writepages 将页提交至块层，完成后页变 Clean，可安全回收。

与前端概念对比：类似前端浏览器的「懒加载 + Virtual DOM Diff」，但本质不同——mmap 的按需分页是 硬件级（MMU + OS 内核） 的物理资源调度，而前端懒加载是 应用级（JS 引擎） 的代码分割；脏页回写类似 React 的 状态批量更新（Batching），但 mmap 是 异步磁盘持久化，涉及 DMA、块层调度与文件系统日志（Journal），而 React 是 内存中的 UI 同步。

### 3. 基础代码与实战验证
```text
/* 极简 mmap 缺页与脏页验证（Linux x86_64，需 gcc 编译） */
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>
#include <string.h>

int main() {
    const char *path = "/tmp/mmap_test.bin";
    int fd = open(path, O_RDWR | O_CREAT, 0644);
    ftruncate(fd, 4096); /* 扩展文件至 1 页（4KB），避免 SIGBUS */

    /* 建立读写共享映射（MAP_SHARED 才会触发回写） */
    char *addr = mmap(NULL, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (addr == MAP_FAILED) { perror("mmap"); return 1; }

    /* 第一次读：触发 major page fault（若页不在 Page Cache） */
    volatile char c = addr[0]; /* CPU #PF → filemap_fault → 读盘 */

    /* 写操作：置位 PTE.D，页进入脏页链表 */
    addr[0] = 'A'; /* 页被标记为 Dirty */

    /* 策略 1：显式同步回写（阻塞直到落盘） */
    msync(addr, 4096, MS_SYNC); /* 立即调用 writepages */

    /* 策略 2：依赖内核异步回写（需等待 dirty_writeback_centisecs） */
    /* 此时若系统内存压力低，页保持 Dirty，由 flusher 线程异步处理 */

    /* 解除映射：内核自动检测 VMA，若有脏页则触发回写 */
    munmap(addr, 4096);
    close(fd);
    return 0;
}

/* 验证命令：strace -e trace=mmap,msync,munmap ./a.out 观察系统调用时序；cat /proc/meminfo | grep Dirty 观察脏页数变化 */
```

### 4. 常见误区与进阶思考
1. 认知误区：mmap 等于「文件已加载到内存」。这是致命误解。mmap 仅建立虚拟地址映射，物理页按需调入；若文件大小远超物理内存，频繁 Page Fault 会导致性能远低于 read/write（陷入内核态开销 + 随机磁盘 I/O）。此外，对 sparse file 或映射范围超过文件实际大小（EOF）的访问会触发 SIGBUS，而非缺页。

2. 进阶思考：在 MAP_SHARED 映射中，若进程 A 与进程 B 同时映射同一文件，进程 A 写入的脏页在尚未回写（仍驻留 Page Cache）时，进程 B 能否立即读到新数据？为什么？（提示：考虑 Page Cache 的唯一性、PTE 的 Dirty 位仅影响回写而非可见性、以及何时需要 msync/msync(MS_INVALIDATE)）
