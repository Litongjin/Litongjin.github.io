---
title: "每日基础技术总结 · 2026-06-13 · 进程地址空间布局：栈、堆、mmap 与 ASLR"
date: 2026-06-13 08:00:00
categories: [技术分享]
tags: ["技术分享", "操作系统基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-06-13 · 进程地址空间布局：栈、堆、mmap 与 ASLR

## 📚 今日主题

> **进程地址空间布局：栈、堆、mmap 与 ASLR**（操作系统基础）

### 1. 核心概念速览
进程地址空间是操作系统为每个进程维护的虚拟内存视图，由 MMU 负责映射到物理页。标准布局（Linux x86-64）从低地址到高地址依次为：代码段（.text）、已初始化数据（.data）、未初始化数据（.bss）、堆（heap，通过 brk/sbrk 向上增长）、内存映射区（mmap，用于文件映射与匿名映射，由 mmap_base 定义，通常向低地址分配）、栈（stack，由 STACK_TOP 定义，向下增长），以及内核保留区。该布局解决三个核心问题：1) 进程间隔离与地址翻译；2) 动态内存分配的策略（brk 与 mmap 的分工）；3) 安全加固（ASLR 对各段基址施加随机偏移，阻止代码重用攻击）。该机制是操作系统、编译器与运行时协同的枢纽；任何涉及 native 代码、Node.js addon、WebAssembly 或性能剖析的工程师都必须理解它，因为内存布局直接决定缓存行为、碎片化、溢出漏洞可利用性和程序启动过程。

### 2. 底层原理剖析
在 Linux 内核加载 ELF 可执行文件时，会依次完成以下步骤：解析段（PT_LOAD）并将它们映射到基址，然后调用 setup_arg_pages 设置栈顶，arch_pick_mmap_base 计算 mmap 基址，randomize_stack_top 和 mmap_rnd 注入熵。布局的数值关系可概括为：
- 栈：初始 rsp 指向 STACK_TOP - random_offset，随后每个 call 将返回地址压栈，rsp 递减；通过帧指针（RBP）链式记录调用上下文。
- 堆：初始 brk 指向 BSS 之后，malloc 对小于 MMAP_THRESHOLD（默认 128KB）的请求通过 sbrk/brk 扩展堆顶；对大请求使用 mmap，在 mmap 区域返回一块匿名映射。
- mmap 区域：mmap_base 通常由 TASK_SIZE - stack_guard - random 计算；共享库、文件映射、匿名映射都在此区域按地址递减方向分配。ASLR 通过 get_random_long 生成 8-28 位的随机偏移，实际熵取决于 arch_mmap_rnd。
- 对比前端：JavaScript 的“调用栈”是 V8 在原生栈之上维护的上下文栈，使用有限大小，但本质是 C++ 对象链；而原生栈是硬件寄存器（RSP/RBP）驱动的线性内存。V8 的堆是垃圾回收堆，位于原生堆段之上，但其对象布局由引擎决定，且使用指针压缩；原生进程堆由 glibc 管理，包含分配元数据和自由链表。理解原生布局能帮助前端工程师调试 Node.js 中的 Buffer 内存、Native Addon 的内存泄漏，以及 WebAssembly 线性内存与主机内存的关系。

### 3. 基础代码与实战验证
```text
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <unistd.h>

int global_var; // .bss 段，位于数据段之后

int main() {
    int local_var; // 局部变量，分配在系统栈上，栈地址通常接近 STACK_TOP
    char *small_heap = malloc(4096);      // 小于 MMAP_THRESHOLD，glibc 优先使用 brk 扩展堆
    char *large_heap = malloc(1024 * 1024); // 超过阈值，glibc 改用 mmap 匿名映射
    char *map = mmap(NULL, 4096, PROT_READ|PROT_WRITE,
                     MAP_PRIVATE|MAP_ANONYMOUS, -1, 0); // 显式请求 mmap 区域

    printf("main:      %p\n", (void*)main);       // 代码段基址（受 ASLR 随机化）
    printf("stack:     %p\n", (void*)&local_var); // 栈变量地址，高地址向下
    printf("small_heap:%p\n", (void*)small_heap); // 堆段地址，紧邻 BSS 向上
    printf("large_heap:%p\n", (void*)large_heap); // 位于 mmap 区域，与栈之间有随机间隔
    printf("mmap:      %p\n", (void*)map);        // mmap 区域内的地址
    printf("\n/proc/self/maps 关键片段:\n");
    system("cat /proc/self/maps | grep -E 'heap|stack|libc|anon' | head -10");

    free(small_heap);
    free(large_heap);
    munmap(map, 4096);
    return 0;
}
```

### 4. 常见误区与进阶思考
常见误区与进阶思考：
误区 1：所有 malloc 分配都来自堆段。实际上，glibc 的 malloc 对大于 MMAP_THRESHOLD（默认 128KB）的请求会使用 mmap 匿名映射，返回地址位于 mmap 区域而非 brk 堆。因此，通过指针地址大小判断分配类型是错误做法，必须结合 /proc/self/maps 或 malloc_info。
误区 2：栈地址固定且一定向下增长。由于 ASLR，栈顶在每次启动时都有随机偏移；同时，部分 ISA（如 HP PA-RISC）的栈增长方向并非向低地址。程序不应依赖绝对地址，应使用相对寻址。
思考题：多次运行上述程序，记录 large_heap 与 stack 的地址，统计其高位随机部分与页内偏移（低 12 位）。请解释为什么 ASLR 只能随机化基址而不能随机化页内偏移，并讨论攻击者如何利用这一性质绕过 ASLR（例如通过侧信道或部分覆盖）。
