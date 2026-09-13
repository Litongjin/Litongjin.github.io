---
title: "每日基础技术总结 · 2026-09-13 · 进程地址空间布局：栈、堆、mmap 与 ASLR"
date: 2026-09-13 08:00:00
categories: [技术分享]
tags: ["技术分享", "操作系统基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-13 · 进程地址空间布局：栈、堆、mmap 与 ASLR

## 📚 今日主题

> **进程地址空间布局：栈、堆、mmap 与 ASLR**（操作系统基础）

### 1. 核心概念速览
进程地址空间是操作系统为每个进程提供的虚拟内存线性视图，由内核 mm_struct 维护，并通过 vm_area_struct 集合描述各个区域。栈用于函数调用帧（局部变量、返回地址、寄存器上下文），从高地址向低地址增长；堆是 data segment 的动态扩展区，通过 brk 系统调用向上增长；mmap 在栈与堆之间的空闲区域建立匿名或文件映射，用于共享库、大块动态内存和模块加载；ASLR 随机化各区域的加载基址，提高攻击者利用内存漏洞的成本。它解决了内存隔离、动态链接、碎片管理和安全缓解问题，是 CPU 分页机制、内核 VMA 管理、glibc 内存分配器共同作用的结果。专业工程师必须掌握它，因为任何语言的运行时、服务端进程调优、容器内存限制、崩溃分析和漏洞挖掘最终都会落在这张地址图上；AI 推理框架的显存分配和 CUDA context 同样依赖虚拟内存映射。

### 2. 底层原理剖析
在 x86-64 Linux 中，用户态虚拟地址空间范围为 0x0000000000000000 至 0x00007fffffffffff（47 位规范地址），内核占用高半区。进程启动后，内核按 ELF 加载器逐步构建 mm_struct：只读代码段映射到低地址，紧接着数据段和 BSS；然后 brk 初始地址向上作为堆起点；栈基址在接近用户空间顶部的位置随机确定，rsp 从这里向下增长；mmap 区域从栈下方的 mmap_base 向下分配。这些区域都用 vm_area_struct 描述，并插入进程的红黑树中。

核心机制：1) brk 系统调用调整 mm->brk 来扩展/收缩堆；glibc malloc 对小于 MMAP_THRESHOLD（通常 128KB）的请求使用堆，超过阈值直接使用 mmap 匿名映射。2) mmap 系统调用在地址空间中查找足够大的空洞，创建新的 vm_area_struct；共享库被 ld.so 以连续多个 vma 映射进该区域。3) ASLR 由内核在 exec 时生成随机量：stack_rand 加到栈顶、brk_rand 加到堆起点、mmap_rand 加到 mmap_base。用户可通过 /proc/sys/kernel/randomize_va_space 控制，2 表示完全开启。

与前端概念的异同：进程地址空间是硬件页表与内核数据结构共同维护的运行时事实，任何指针逃逸都无法绕过；JS 引擎中的堆栈是引擎在系统内存之上实现的 GC 对象图，JS 代码看不到真实地址。TS 的接口是编译期类型约束，编译后删除；Java 的接口是 JVM 类型系统的一部分，在字节码中真实存在。前者是‘机制上的硬约束’，后者是‘抽象层的协议’，理解层级差异就能避免把语言层替换和 OS 层内存布局混为一谈。

### 3. 基础代码与实战验证
```text
验证程序（纯 C，无框架）：

#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>

int main() {
    int stack_var; // 栈变量: 由 rsp 相对寻址，位于栈框架内，高地址且随 ASLR 变化
    void *heap_val = malloc(64); // 小分配: glibc 通过 brk 调整堆顶，地址在 mmap 区域之下
    void *map_val = mmap(NULL, 4096, PROT_READ | PROT_WRITE,
                         MAP_PRIVATE | MAP_ANONYMOUS, -1, 0); // 匿名映射: 内核在 mmap_base 向下找空洞
    printf("stack=%p heap=%p mmap=%p main=%p\n",
           (void *)&stack_var, heap_val, map_val, (void *)&main); // 输出四个区域的虚拟地址
    munmap(map_val, 4096);
    free(heap_val);
    return 0;
}

编译：gcc -o addr addr.c
运行：./addr  连续执行两次，可见 stack/heap/mmap/main 地址均带随机偏移。
对比：sudo sysctl -w kernel.randomize_va_space=0 后再次运行，地址不再变化，证明偏移全部来自 ASLR。
注意：现代 GCC 默认生成 PIE 可执行文件，所以 main 地址也参与随机化；小 malloc 走 brk 堆，尝试 malloc(1024*1024) 会发现返回地址来自 mmap 区域。
```

### 4. 常见误区与进阶思考
常见误区：
误区一：认为栈一定固定在高地址、堆固定低地址。实际上这些基址由 ASLR 每次 exec 随机化，同一进程的不同次运行地址完全不同；栈地址的高低位只是布局策略，不是硬件保证。把绝对地址写入跨进程持久化数据（如序列化指针）必然不可靠。

误区二：认为所有动态内存都来自堆（brk 区）。glibc 有 MMAP_THRESHOLD，默认超过 128KB 的 malloc 直接调用 mmap；多线程 malloc 的 arena 本身也由 mmap 创建，再在内部按 bin 切分。因此看到地址落在堆和 mmap 两个明显不同的区域是正常的，不能简单用堆概念概括所有动态分配。

深度思考题：64 位 Linux 上 mmap 匿名映射地址与栈地址之间会保留巨大的空洞，这个空洞为何存在？如果完全关闭 ASLR，mmap_base 和 brk 之间又靠什么机制避免 VMA 重叠？请从内核 unmapped_area 查询逻辑、mmap_rnd_bits 以及进程文件映射栈区的增长方向推演。
