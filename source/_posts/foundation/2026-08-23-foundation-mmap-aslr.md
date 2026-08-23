---
title: "每日基础技术总结 · 2026-08-23 · 进程地址空间布局：栈、堆、mmap 与 ASLR"
date: 2026-08-23 08:00:00
categories: [技术分享]
tags: ["技术分享", "操作系统基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-23 · 进程地址空间布局：栈、堆、mmap 与 ASLR

## 📚 今日主题

> **进程地址空间布局：栈、堆、mmap 与 ASLR**（操作系统基础）

### 1. 核心概念速览
进程地址空间是操作系统为每个进程提供的虚拟内存抽象，使进程认为自己独占连续的内存区域。其标准布局（x86-64 Linux 为例）从高地址到低地址依次为：内核空间、栈（向下增长）、mmap 区域（文件映射与匿名映射）、堆（向上增长）、BSS、数据段、文本段。核心机制是虚拟内存 + 页表转换，由内核管理，配合 MMU 将虚拟地址映射到物理页帧。它解决的问题是：内存隔离（进程间互不干扰）、地址空间连续性（物理内存碎片化对进程透明）、按需加载（只有被访问的页才装入物理内存）。ASLR（地址空间布局随机化）则在每次进程启动时随机化栈、堆、mmap 基址及动态库加载地址，使攻击者无法预判目标内存地址，是缓解缓冲区溢出等内存破坏攻击的关键安全机制。在整个计算机体系中，它是操作系统、编译原理、体系结构三者的交汇点：编译器生成的可执行文件依赖该布局，CPU 的分页机制实现该布局，安全攻防围绕该布局展开。专业工程师必须掌握它，因为定位段错误、分析内存泄漏、理解安全漏洞（如栈溢出、堆溢出、ret2libc）、优化内存分配性能，都直接建立在对该布局的精确理解之上；对于转向 AI/后端，理解容器内存限制、Go/JVM 运行时内存管理也与之一脉相承。

### 2. 底层原理剖析
一、地址空间分层结构（以 64 位 Linux 为例）。虚拟地址从 0x0000000000000000 到 0xffffffffffffffff，内核通常占据高地址区（如 0xffff800000000000 以上），用户态从低地址开始：文本段（ELF 头部、代码段 .text，只读）、数据段（.data 已初始化全局变量，.bss 未初始化全局变量）、堆（由 brk 系统调用扩展，起始于数据段之上，向上增长）、mmap 区域（从高地址向下映射，用于共享库、文件映射、大块 malloc）、栈（位于用户态最高地址附近，向下增长，栈顶由 rsp 寄存器指向）。栈与堆相向生长，mmap 区域插入两者之间，隔离动态分配与函数调用。
二、栈机制。函数调用时，CPU 将返回地址压栈，然后通过 push/pop 和 rbp/rsp 寄存器维护栈帧。局部变量、函数参数、返回地址都位于栈中。栈向下增长意味着低地址是栈的'顶'，高地址是栈的'底'。每次函数调用压栈，rsp 减小；返回时恢复 rsp 并弹出返回地址。栈溢出（如递归过深）会导致 rsp 越过栈的红色区域（red zone）甚至触碰到 mmap 区域，引发段错误。
三、堆机制。传统 Unix 使用 brk/sbrk 系统调用将数据段末尾（program break）向上移动来分配堆内存。但现代 glibc 的 malloc 策略是：小于 MMAP_THRESHOLD（默认 128KB）的分配通过 brk 扩展堆；大于该阈值时改用 mmap 创建匿名映射，放在 mmap 区域，释放时直接 munmap。因此'堆'实际包含两部分：由 brk 管理的连续堆区，以及由 mmap 管理的离散大块分配区。这避免了堆碎片化，也让大块内存能独立释放回内核。
四、ASLR 机制。内核在 execve 加载程序时，以熵值随机化栈基址、mmap 基址、堆起始地址、动态链接器加载地址。文本段基址（ET_EXEC 可执行文件）通常固定，但 PIE（位置无关可执行）也会随机化。随机化发生在页粒度（如 4KB 对齐），且各区域独立偏移。ASLR 不会改变布局的相对顺序（栈仍在最高区，堆仍在低区），但每次运行的具体地址不同。可通过 /proc/sys/kernel/randomize_va_space 控制（0 关闭，2 全开）。
五、与前端概念的对比。前端工程师熟知的 JavaScript 执行模型有'调用栈'和'堆（对象堆）'，但那是语言运行时（如 V8）内的抽象：调用栈存放基本值和函数调用帧，对象堆存放引用类型，由 GC 管理。而本主题的栈和堆是操作系统层面的虚拟内存区域，JS 的'栈'最终也映射到进程地址空间的栈区，JS 的'对象堆'则对应进程堆或 mmap 区域。类比 Java 的接口与 TS 的接口：两者同名但语义不同（Java 接口是运行时多态契约，TS 接口是编译期结构约束）；同理，前端谈的'栈'是语言运行时控制流结构，OS 的'栈'是硬件寄存器与内存布局共同定义的执行上下文存储区，底层机制完全不同。

### 3. 基础代码与实战验证
```text
以下代码用 C 语言验证地址空间布局，编译后运行可观察各区域地址分布。

#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <unistd.h>

int global_data = 42;          // 数据段（.data）
int global_bss;                // BSS 段（未初始化）

void func() {
    int local = 0;             // 栈变量
    printf("stack local      : %p\n", (void*)&local);
}

int main() {
    static int static_var = 1; // 仍位于数据段，非栈
    int *small_heap = malloc(16);          // 小于阈值，走 brk，位于堆区
    int *large_heap = malloc(128 * 1024);  // 大于阈值，走 mmap，位于 mmap 区域
    int *mmap_ptr = mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);

    printf("text  main       : %p\n", (void*)main);        // 文本段低地址
    printf("data  global_data : %p\n", (void*)&global_data); // 数据段
    printf("bss   global_bss  : %p\n", (void*)&global_bss);  // BSS
    printf("heap  small(16)   : %p\n", (void*)small_heap);    // 堆区（brk）
    printf("heap  large(128K) : %p\n", (void*)large_heap);    // mmap 区域（高地址）
    printf("mmap  anon        : %p\n", mmap_ptr);            // mmap 区域
    func();  // 打印栈地址
    printf("stack main local  : %p\n", (void*)&static_var); // 注意 static_var 在数据段

    free(small_heap);  // 释放小堆块：缩小 brk（可能不立即还回内核）
    free(large_heap);  // 释放大堆块：调用 munmap，立即解除映射
    munmap(mmap_ptr, 4096);
    return 0;
}

关键注释：
- (void*)main 取函数指针地址，text 段位于最低地址（0x400000 附近，非 PIE）或随机地址（PIE 开启时）。
- small_heap 通过 malloc 内部调用 brk，地址在 .bss 之后且向上增长，通常与全局变量地址相差较小。
- large_heap 因为大于 MMAP_THRESHOLD，glibc 直接调用 mmap，地址位于 mmap 区域（栈下方较远处），与 small_heap 有明显差距。
- 每次运行，除文本段（非 PIE）外，栈、堆、mmap 地址都会因 ASLR 变化；可通过 echo 0 | sudo tee /proc/sys/kernel/randomize_va_space 关闭 ASLR 验证固定布局。
- 注意 static_var 声明在 main 内但存储期是静态的，它位于数据段，不是栈，这一点常被误解。
```

### 4. 常见误区与进阶思考
误区一：认为 malloc 分配的内存全部在堆区。实际上 glibc 对大于 MMAP_THRESHOLD 的分配使用 mmap，位于 mmap 区域而非传统 brk 堆。这会带来行为差异：大块内存释放后立即归还内核，而小块内存可能仍留在进程堆中形成碎片。调试或分析内存占用时，不能仅凭分配大小推断区域，应通过 malloc_usable_size 和 /proc/<pid>/maps 确认。
误区二：混淆虚拟地址与物理地址。进程地址空间中的地址是虚拟地址，经过页表转换才到物理地址。ASLR 随机化的是虚拟地址布局，并不影响物理页的分配位置。因此，即使两个进程的相同虚拟地址指向不同物理页，或者同一物理页被映射到多个虚拟地址，都是正常现象。若在代码中直接使用地址值比较或强制转换，会因 ASLR 和页表映射而失效，这也是为什么必须通过 API 而非硬编码地址操作内存。
思考题：ASLR 随机化的是进程地址空间中哪些区域的基址？如果关闭 ASLR，缓冲区溢出攻击（如 ret2libc）需要知道哪些固定地址？请结合页表机制说明：为什么 ASLR 无法防止攻击者利用堆喷射（heap spraying）来绕过随机化？要回答清楚，必须理解 mmap 区域在堆喷射中如何被大量占用，以及攻击者如何通过暴力猜测或信息泄露来覆盖随机熵。
