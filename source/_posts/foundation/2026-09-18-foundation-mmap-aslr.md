---
title: "每日基础技术总结 · 2026-09-18 · 进程地址空间布局：栈、堆、mmap 与 ASLR"
date: 2026-09-18 08:00:00
categories: [技术分享]
tags: ["技术分享", "操作系统基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-18 · 进程地址空间布局：栈、堆、mmap 与 ASLR

## 📚 今日主题

> **进程地址空间布局：栈、堆、mmap 与 ASLR**（操作系统基础）

### 1. 核心概念速览
## 1. 核心概念速览

**定义**
进程地址空间（process address space）是内核为单个进程建立的虚拟地址范围模型，由 task_struct->mm（struct mm_struct）持有。x86-64 上用户态范围为 0x0000_0000_0000_0000 ~ 0x0000_7fff_ffff_ffff，内核态位于 0xffff_8000_0000_0000 以上的上半区，二者由页表项的权限位与 MMU 检查隔离。用户态部分不是同质内存，而由若干个互不重叠的 VMA（vm_area_struct）描述，每个 VMA 定义：区间 [vm_start, vm_end)、权限（VM_READ/VM_WRITE/VM_EXEC/VM_SHARED）、后备对象（vm_file + vm_pgoff，匿名映射为空）、操作集（vm_ops）。这些 VMA 以 vm_start 为键组织在红黑树中，mmap/brk/mprotect 都是对 VMA 集合的插入、合并、分裂操作。

关键结论：栈、堆、mmap 区不是三类物理上不同的内存，而是三类 VMA 的来源与增长策略差异。

- 栈：execve 时由内核建立，起始于随机化的 stack_top，向低地址按缺页自动扩张，受 RLIMIT_STACK 与 guard gap 约束。
- 堆：brk/mmap 的产物。brk 把 data 段末端（program break）向高地址推进，语义上是「单一连续区间」；glibc malloc 在这之上实现 chunk/bin 子分配。
- mmap 区：由 mmap(2)/mmap2 显式建立，含文件映射（动态库、权重文件）与匿名映射（大块 malloc、线程栈、JIT 代码、WASM 线性内存预留）。

其间还夹着 vDSO/vvar（内核注入的用户态映射，用于加速 gettimeofday 等调用）。

**它解决什么问题**
1. 隔离：进程只能访问自己页表中已建立映射的虚拟地址，物理内存无需连续、可超配与复用。
2. 按需分页（demand paging）：VMA 登记阶段不分配物理页，首次访问触发 page fault 才建立 PTE；匿名页先统一映射到只读零页 + COW。
3. 共享与零拷贝：文件映射可让多进程共享同一 page cache 页，避免 read/write 的用户态-内核态数据拷贝。
4. 抽象：为 malloc、GC、JIT、线程栈实现提供统一的「可增长虚拟区间」原语。

**在体系中的位置**
向下：MMU 四级页表（PGD→PUD→PMD→PTE）、TLB、物理页帧分配器、缺页异常处理、反向映射 rmap。向上：libc malloc / tcmalloc / jemalloc、语言运行时（V8 Heap、Go runtime arena、JVM 堆）、动态链接器 ld.so、线程栈，以及安全缓解体系（ASLR / NX / PIE / RELRO）。
AI 栈同样处处依赖它：mmap 加载大模型权重（llama.cpp 默认把权重文件作为文件映射 VMA，按页 fault 进 page cache，多进程可共享同一份物理页）、CUDA pinned memory、共享内存 IPC、Zero-copy 传输，全部建立在 VMA 语义之上。

**为什么专业工程师必须掌握**
- 只会读 heapUsed 而不看 VSZ/RSS 的人无法定位定位 OOM：V8 堆是用户态分配器层，容器 OOM 判定依据是 RSS。
- 不区分 brk 与 mmap 的分界（mmap_threshold），就解释不了「free 之后 RSS 不下降」「首次分配大对象慢」「进程 VSZ 极大但 RSS 很小」这些现象。
- 不掌握 ASLR 的随机化粒度与「只随机化基址」这一性质，就无法理解 PIE/CET/Canary 为何必要、一次 info leak 为何致命。

### 2. 底层原理剖析
## 2. 底层原理剖析

### 2.1 x86-64 Linux 用户态布局（低地址 → 高地址）

0x0000000000000000  非映射区（mmap_min_addr 阻止低地址 mmap，捕获 NULL 解引用）
        ↓
E L F 映像（PIE 时为随机 load_bias 起始）
  .text / .rodata（R-X）→ .data / .bss（RW-）
        ↓
program break（brk 起点，随机化窗口 32MB）
  heap（RW-，向高地址增长；glibc 主 arena）
        ↓        空洞（heap 向上增长、mmap 区向下增长，二者争夺同一空洞）
mmap 区（mmap_base 随机，向下增长）
  文件映射：libc.so / ld.so / 大权重文件
  匿名映射：大块 malloc、pthread 栈、JIT、WASM 预留
        ↓
  栈（stack_top 随机，向低地址增长，RLIMIT_STACK 默认 8MB，栈底有 guard page）
        ↓
[vvar][vdso]（内核注入，R-X）
        ↓
0x00007fffffffffff  用户态上界

32 位布局不同：mmap 区与栈共享同一个向上增长的空洞，所以「栈与堆相向增长最终碰撞」是 32 位语境的经验；x86-64 下堆与栈之间隔着 mmap 区，先耗尽的是 mmap_base 与 program break 之间的空洞（表现为 mmap 返回 ENOMEM），或撞到 RLIMIT_STACK 与 guard page。

### 2.2 建立与访问：VMA → 页表 → 物理页

VMA 只是「意图声明」，页表才是「实际映射」。访问一个虚拟地址 va 的完整路径：

1. MMU 以 CR3 指向的 PGD 为根逐级查表（PGD→PUD→PMD→PTE）；命中 TLB 则直接完成翻译。
2. 若 PTE 的 Present 位为 0 或权限不符（如对只读页写入）→ 触发 #PF，CPU 把出错地址写入 CR2，并把错误码（P/W/U/RSVD）压栈。
3. 内核 do_page_fault → find_vma(mm, va)：按红黑树查覆盖 va 的 VMA。
4. 若 va 不落在任何 VMA 内（落在空洞）→ 非法访问 → 投递 SIGSEGV（默认终止并 core dump）。这是 null 解引用、野指针、栈溢出撞 guard page 的统一来源。
5. 若 VMA 有效但 PTE 缺失，按 VMA 类型分派：
   - 匿名 VMA：分配物理页 → 清零 → 填 PTE；若是 MAP_PRIVATE 且父页存在则走 COW。
   - 文件 VMA：在 page cache 中查找或读入 vm_pgoff + 页偏移对应的页，建立 PTE；MAP_PRIVATE 时标只读，写入触发 COW。
   - 栈 VMA 且 va 处于允许扩张范围：先 expand_stack 扩大 VMA，再分配页；超出则 SIGSEGV。
6. 返回用户态，重新执行那条触发 #PF 的指令（不是重试系统调用，而是硬件级重试）。

这就是「首次访问比后续访问慢一个数量级」的根因：一次 fault 涉及异常入口、红黑树查找、页帧分配、清零（memset 4KB）、页表写入与 TLB 刷新。

### 2.3 brk 与 mmap 的分工

维度 | brk | mmap(MAP_ANONYMOUS)
语义 | 移动 data 段末端的 program break，单一连续区间 | 新建独立 VMA
系统调用 | brk(2)；glibc 在阈值内批量预订，减少调用 | mmap(2) / munmap(2)，每次两个调用
增长方向 | 向高地址 | 内核从 mmap_base 向下寻找空洞
归还能力 | 仅能收缩 top chunk；内部碎片无法归还，RSS 常不下降 | munmap 立即解除映射，RSS 立刻下降
适用场景 | glibc 主 arena 的中小分配 | 超过 mmap_threshold（默认 128KB，且按释放情况动态调整）、pthread 线程栈、显式映射
代价 | 无 VMA 抖动、无额外陷入；碎片不可回收 | 系统调用 + TLB shootdown；无碎片、可精确回收

### 2.4 ASLR 的实现机制

execve 阶段内核一次性随机化的对象：
- PIE 可执行文件与共享库的 load_bias：ELF 整体基址随机（x86-64 默认 28 位、页对齐，受 ELF_ET_DYN_BASE 约束）。
- mmap_base：由 /proc/sys/vm/mmap_rnd_bits（x86-64 默认 28）决定匿名/文件映射区的起始高度。
- stack_top：randomize_stack_top() 使用 STACK_RND_MASK，x86-64 为 22 位页粒度（约 16GB 窗口）。
- brk 起点：randomize_page(mm->start_brk, 0x02000000)，13 位页粒度（32MB 窗口）。
- vDSO/vvar 位置同样随机化。

控制开关：/proc/sys/kernel/randomize_va_space，0=关闭；1=仅 mmap/stack/vdso；2=全开（含 brk 与 PIE 基址）。单进程可用 personality(ADDR_NO_RANDOMIZE) 或 setarch -R 关闭，无需 root 改全局 sysctl。

最关键的性质：ASLR 只随机化「各区域的基址」，模块内部符号的相对偏移在链接期完全固定。因此任意一次地址泄漏（info leak）即可反推基址，使 ASLR 对该攻击者退化为零熵。ASLR 必须与 PIE、Full RELRO、Canary、CET 组合，不能单独构成防线。

### 2.5 与前端已有知识体系的对照

1) 「内存」是两层抽象的叠加。
   - 用户态分配器层：V8 的 Page Allocator / Semi-space / Large Object Space，或 glibc 的 arena/chunk/bin，自行记账、自行复用、自行决定向 OS 要多少。
   - 内核层：VMA + 页表，只按整页管理，完全不理解分配器语义。
   前端熟悉的 process.memoryUsage() 中，heapTotal/heapUsed 属于第一层，rss/external/arrayBuffers 属于第二层（更接近内核视角），两者永远不对应 —— 这与 glibc 的 malloc_stats 与 RSS 不对应完全同构。

2) V8 Pointer Compression Cage。V8 启动时一次性向 OS 预留 4GB 虚拟区间（mmap PROT_NONE），把对象指针压缩为 cage 内 32 位偏移。这正是「保留型 VMA」用法：虚拟地址保留、物理页按需提交，与进程地址空间中 mmap 区的语义完全一致；cage 基址自身随机化，提供少量熵。

3) WebAssembly 线性内存。ArrayBuffer / WebAssembly.Memory 是 4GB 虚拟预留 + memory.grow 只增不减，语义等价于 brk 的「单调增长上界」模型；而 JS 的 ArrayBuffer 一旦 detach 便不可复用，与 munmap 后的重新映射不同。

4) 前端的结构性缺失。浏览器进程与 Node 主进程中，栈与堆的划分、VMA 组织、ASLR 全部由 OS 与运行时代管，前端工程师极少直接面对地址空间布局这一层。一旦写 Node N-API 原生扩展、调优大内存服务、分析容器 OOM，这一层立刻成为瓶颈知识。

### 3. 基础代码与实战验证
```text
以下程序不做任何抽象，直接打印各类对象在虚拟地址空间中的位置，并读出内核视角的 VMA 列表做对照。

编译与运行：
  gcc -O0 -g -fPIE -pie   layout.c -o layout_pie
  gcc -O0 -g -no-pie      layout.c -o layout_nopie
  ./layout_pie            # 连续执行三次，观察地址变化
  ./layout_nopie          # 连续执行三次，观察 .text/.data 是否固定
  setarch $(uname -m) -R ./layout_pie     # 对单个进程关闭 ASLR
  cat /proc/sys/kernel/randomize_va_space # 系统级策略

源码：

#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <unistd.h>

int g_data = 1;          /* .data：已初始化全局变量，位于 ELF 映像 VMA 内，相对基址偏移链接期固定 */
int g_bss;               /* .bss：不占文件空间；首次写入触发匿名零页缺页 */
static int s_static = 2; /* .data，偏移在链接期确定 */

int main(void) {
    int   local = 0;                        /* 栈：由 rbp/rsp 相对寻址，落在栈 VMA 内 */
    void *small = malloc(64);               /* 主 arena 的 top chunk，本质是 brk 区间内的子分配 */
    void *big   = malloc(4 * 1024 * 1024);  /* 超过 mmap_threshold，glibc 直接 mmap，创建独立 VMA */
    void *anon  = mmap(NULL, 4096, PROT_READ | PROT_WRITE,
                       MAP_PRIVATE | MAP_ANONYMOUS, -1, 0); /* 只登记 VMA，物理页首次访问才分配 */

    printf("main     %p\n", (void *)main);   /* PIE 下 = load_bias + 固定偏移；no-pie 下为常量 */
    printf("g_data   %p\n", (void *)&g_data);
    printf("g_bss    %p\n", (void *)&g_bss);
    printf("s_static %p\n", (void *)&s_static);
    printf("local    %p\n", (void *)&local);  /* 随 stack_top 随机化，每次运行都变 */
    printf("small    %p\n", small);           /* brk 区间，紧邻 data 段之后 */
    printf("big      %p\n", big);             /* mmap 区，通常与 small 相距极远 */
    printf("anon     %p\n", anon);
    printf("environ  %p\n", (void *)environ); /* 环境变量位于栈顶附近 */

    FILE *f = fopen("/proc/self/maps", "r"); /* 内核视角 VMA 列表，格式：start-end perms offset dev inode path */
    char line[256];
    while (fgets(line, sizeof line, f))
        fputs(line, stdout);
    return 0;
}

关键观察点（用输出验证上文机制）：
1. layout_pie 连跑三次：main / local / small / big / anon 全部变化；layout_nopie 连跑三次：main 与 g_data 恒定，仅栈与 mmap 区变化。这直接证明 ASLR 随机化的是各区域基址，而 PIE 决定代码段是否可被随机化。
2. small 与 &g_bss 通常相差一个页以内且位于更高地址：说明它来自 brk 的连续区间；其差值不是 64，而是 chunk 头 + 对齐后的尺寸，证明 malloc 是分配器层子分配，不是系统调用直通。
3. big 与 anon 落在同一高地址带（0x7f...），即 mmap 区；对 big 调用 free 后 RSS 立即下降，对 small 调用 free 后 RSS 常不下降 —— 对应 brk 无法随意收缩的限制。
4. /proc/self/maps 中每一行就是一个 VMA。把 [heap]、[stack]、匿名段、libc.so 的文件映射段与前述打印地址逐一对应，即可确认「三类内存 = 三类 VMA 来源」。
5. environ 与 local 同处高地址带，且 environ 地址更高，印证栈位于高地址并向低地址增长。
```

### 4. 常见误区与进阶思考
## 4. 常见误区与进阶思考

**误区一：把 malloc/free 的虚拟地址语义等同于物理内存与内核映射语义**
- 表现：认为 malloc 返回连续地址就意味着物理内存连续；认为 free 之后 RSS 必然下降；认为两次 malloc 的地址差等于对象大小。
- 实质：malloc 是用户态分配器，在已存在的 VMA（主 arena 的 brk 区间，或 mmap 出来的独立 arena）内按 chunk 切分，地址连续性只是分配器策略的结果；物理页由内核在缺页时按页帧分配，与虚拟地址顺序无关。free 仅把 chunk 还给分配器的 bin；只有当释放的是 top chunk 且满足收缩条件时 glibc 才调用 brk 归还内核；超过 mmap_threshold 的分配走 mmap，munmap 才立即归还。
- 后果：只看 free 前后的 RSS 判断泄漏会得出错误结论，把分配器缓存误判为内存泄漏，或忽略真正由 mmap 增长导致的 VSZ 膨胀。

**误区二：用 32 位的直觉理解 x86-64 布局与 ASLR 强度**
- 表现：认为栈与堆相向增长终会碰撞；认为栈固定 8MB；认为 ASLR 一旦开启就无法绕过。
- 实质：x86-64 下 mmap 区位于堆与栈之间（mmap_base 在高地址、向下增长），二者并不直接相向；RLIMIT_STACK 的 8MB 是扩张上限而非预分配，栈按缺页逐步扩张，真正的保护是栈底 guard page（栈溢出先撞 guard 得到 SIGSEGV，而非静默覆写相邻 VMA）。ASLR 的熵在不同区域并不相同（x86-64 约：mmap/PIE 28 位、栈 22 位、brk 13 位），且只随机化基址，模块内偏移固定，一次 info leak 即可让它等价于未开启；32 位下 mmap 熵仅 8 位，可被 fork 型服务端爆破。

**思考题**
x86-64 Linux 上 mmap_base 有 28 位随机化（页粒度），栈 22 位，而 brk 起点只有 13 位（32MB 窗口）。请回答：
(a) 为什么堆（brk）的熵被刻意压得远小于 mmap 区？请从 brk 的语义（单一连续区间、与 data 段毗邻、必须为其向上增长保留足够空间以免与 mmap_base 冲突）与内核实现（randomize_page 的窗口约束）两个角度解释，并判定这是实现妥协还是设计取舍。
(b) 若把 glibc 的 MMAP_THRESHOLD 调到 4KB，令绝大多数分配都经 mmap 完成，进程整体的 ASLR 有效熵与「堆对象地址的不可预测性」会如何变化？代价是什么（系统调用开销、VMA 数量与 vm_area_struct 红黑树压力、munmap 抖动、TLB shootdown）？
(c) 由此说明：ASLR 的强度是「所有可达地址中最弱环节」的函数，而不是某个区域熵值的平均。
