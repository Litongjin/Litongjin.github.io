---
title: "每日基础技术总结 · 2026-09-20 · 进程与线程的区别"
date: 2026-09-20 07:02:17
categories: [技术分享]
tags: ["技术分享", "后端基础（Node.js / Java / Python）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-20 · 进程与线程的区别

## 📚 今日主题

> **进程与线程的区别**（后端基础（Node.js / Java / Python））

### 1. 核心概念速览
定义（以 Linux 内核语义为准，脱离教科书的拟人化表述）：

- 进程（process）：内核资源分配的隔离边界。一个进程 = 一个独立虚拟地址空间（mm_struct + 页表）+ 文件描述符表 + 文件系统上下文（cwd/umask）+ 信号处理函数表 + 凭证 + 资源限额（rlimit），以及至少一个 task_struct。它解决的问题是隔离：因为 MMU 强制每次访存都经过页表翻译，A 进程的指针值在 B 进程的页表下可能根本不映射或映射到完全不同的物理页，所以进程间不存在任何隐式数据通道。
- 线程（thread）：内核调度的执行边界。一个线程独占 CPU 寄存器组、内核栈、用户栈、TLS、errno、信号掩码；与同进程内其他线程共享地址空间、fd 表、信号处理表、cwd、rlimit。它是 CPU 时间片分配的最小对象。

本质：Linux 内核根本没有进程/线程这两种类型，二者都用 task_struct 表示，差异只由 clone(2) 的 flags 决定——CLONE_VM 共享 mm_struct、CLONE_FILES 共享 fd 表、CLONE_FS、CLONE_SIGHAND、CLONE_THREAD 加入同一线程组（tgid 相同）、CLONE_SETTLS。pthread_create 就是带这一组 flag 的 clone；fork 就是不带这些 flag 的 clone。所谓「进程」只是 tgid 相等的一组 task 的外在视图，「进程是资源分配单位」是概念抽象而非内核实体。

为什么必须掌握：
1. 它是所有并发运行时的物理底座。Node 的 event loop + libuv 线程池、Java 21 之前的 platform thread 与 JEP 444 virtual thread、Python 的 GIL、Go 的 GMP、浏览器 renderer 进程模型，全部是在 OS 这两级抽象之上的封装。不理解 clone flags，就无法解释 worker_threads 与 cluster 的成本差异，也无法解释为什么 Node 的普通对象跨 worker 必须序列化。
2. 它直接决定故障域与共享成本，从而决定架构：多进程 = 强隔离 + IPC 序列化/拷贝成本；多线程 = 零拷贝共享 + 锁与内存序成本 + 无故障隔离。
3. 它是云原生语义的基础：容器 = namespace（隔离视角）+ cgroup（资源限额），被隔离的实体仍是进程/线程组；K8s 调度 Pod（一组共享 namespace 的进程）。
4. AI 工程侧：CUDA context 以进程为粒度，多进程训练与 DataLoader num_workers 绕开 GIL，NCCL 的进程/线程通信拓扑选择，都建立在这一层之上。

### 2. 底层原理剖析
一、内核数据结构（task_struct 的关键字段）
- mm / active_mm：指向 mm_struct（地址空间）。CLONE_VM 时多个 task 指向同一 mm_struct —— 这就是「线程共享内存」的全部真相。
- files、fs：fd 表与 fs 上下文，CLONE_FILES / CLONE_FS 决定是否共享。
- sighand、blocked：信号处理函数表与信号掩码，线程各自维护掩码。
- pid、tgid：tgid 相等即同一线程组。getpid() 返回 tgid，gettid() 返回 pid；/proc/<tgid>/task/<tid> 即由此组织，/proc/<pid>/task 下的目录数 = 线程数。
- 内核栈（通常 8KB/16KB，每线程独立）、thread_info、栈指针切换位置。
- se / vruntime / prio：CFS 调度实体。调度器只看 task，不看它属于哪个进程，因此同一进程的线程之间也在互相抢占 CPU。

二、创建路径与开销的真实来源
fork() → clone(SIGCHLD, 0)：复制 mm_struct 等元数据，父子页表项复制后共同指向同一物理页并清除可写位（COW 只延迟物理页复制，不延迟页表与元数据复制）。任一方写入触发 page fault，内核分配新物理页、复制内容、重建 PTE 并恢复可写。
execve()：替换 mm_struct 与用户态上下文，是唯一真正更换「程序映像」的系统调用（child_process.fork 之后加载 V8 走的正是这条路）。
pthread_create() → clone(CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD|CLONE_SETTLS)：不复制页表，只 mmap 用户栈 + 分配内核栈。
量级：Linux 上 fork 因 COW 与 clone 同量级（微秒级），真正的差距在上下文切换与故障域，而不是创建本身。

三、上下文切换的精确步骤
1) 保存当前线程的 callee-saved 寄存器（rbx/rbp/r12-r15/rsp）到其内核栈；
2) 把内核栈指针切到目标 task；
3) 若 mm 不同（不同进程）：写 CR3 装载新页表基址，TLB 大范围失效（PCID/ASID 打标签可缓解，避免全局 flush）；
4) 恢复目标寄存器与用户栈，sysret/iret 返回用户态。
结论：同进程线程切换不换 CR3，TLB 与 cache 局部性好，显著便宜；跨进程切换有 CR3 写入 + TLB 污染 + cache 冷启动成本。但即便线程切换也要陷入内核，量级 1–5us，远高于函数调用（ns）。

四、用户态线程模型（前端工程师最需要补的一层）
- 1:1：pthread、Java platform thread、Node worker_threads，一个用户线程对应一个 task_struct，可利用多核；
- N:1：绿色线程，任一线程阻塞式系统调用会阻塞整个进程，无法利用多核；
- M:N：Go goroutine、Java virtual thread、Erlang，运行时在少量内核线程上复用大量用户态协程，阻塞点由运行时拦截并让出。

五、与前端已有概念对照
- Web Worker ≈ 线程模型但被刻意做成弱共享：同进程、同地址空间，但禁止访问 DOM；默认传值走 structured clone（深拷贝，语义上接近 IPC 序列化）；只有 SharedArrayBuffer + Atomics 才提供真正的共享内存与内存序语义。
- 跨站 iframe（Chrome Site Isolation 下）≈ 进程模型：独立地址空间与渲染上下文，通信必须经过浏览器进程的 IPC，postMessage 底层仍是序列化。
- 浏览器把每个 site 放独立 renderer 进程，与 Node 默认单进程多线程（event loop + libuv 默认 4 线程池 + 可选 worker_threads）是两种权衡：前者用进程换安全隔离与崩溃域收敛（一个页面 OOM 不拖垮全浏览器），后者用线程换零拷贝共享与启动/内存开销。
- Node 的 cluster（fork 子进程，各自独立 V8 heap 与 GC，默认由 master round-robin 分发连接句柄）vs worker_threads（同进程多 isolate，每个 worker 有独立 event loop 与堆，只有 SAB 是共享的）——这正是 fork 与 pthread_create 在运行时层的投影。

### 3. 基础代码与实战验证
```text
验证 1（Node 内置模块，零依赖，需 Linux 观察 /proc）：

const { Worker, isMainThread, workerData, threadId } = require('worker_threads');
const { spawn } = require('child_process');
const fs = require('fs');

// /proc/self/task 下每个目录对应一个 task_struct，目录数即线程组大小
// 主线程 + libuv 线程池通常已 > 1；worker 启动后该值会再增加
console.log('[pid=%d tid=%d] threads=%d', process.pid, threadId, fs.readdirSync('/proc/self/task').length);

// ---------- 1) 线程：共享同一 mm_struct，写同一物理页 ----------
const sab = new SharedArrayBuffer(4); // 内核分配物理页，被多个线程映射
if (isMainThread) {
  const view = new Int32Array(sab);
  view[0] = 0;
  const w = new Worker(__filename, { workerData: sab });
  w.on('exit', () => {
    // 42：worker 的写入对主线程立即可见，因为两边页表指向同一物理页
    console.log('[main] view[0] =', view[0]);
    console.log('[main] worker 的 process.pid 与主线程相同（同 tgid），仅 threadId 不同');
  });
} else {
  // 注意：workerData 中的普通对象会被 structured clone 深拷贝，只有 SAB 是共享的
  const view = new Int32Array(workerData);
  Atomics.store(view, 0, 42); // 共享内存上的写必须用 Atomics 才有确定的内存序与可见性
}

// ---------- 2) 进程：独立 mm_struct，虚拟地址相同但物理页不同 ----------
const code = [
  'globalThis.counter = 1;', // 只写进子进程私有的地址空间
  'console.log("[child]  pid=%d ppid=%d counter=%s", process.pid, process.ppid, globalThis.counter);',
].join('\n');
const child = spawn(process.execPath, ['-e', code], { stdio: 'inherit' });
child.on('exit', () => {
  // undefined：父子 task_struct 指向不同 mm_struct，地址空间不共享，任何对象都需经 IPC 序列化
  console.log('[parent] counter =', globalThis.counter);
});

验证 2（C 语言，直击 clone flags 与 COW 的差异，gcc t.c -lpthread）：

#include <stdio.h>
#include <unistd.h>
#include <pthread.h>
#include <sys/wait.h>

int g = 1; // .data 段，是地址空间的一部分

void *thr(void *_) { g = 42; return NULL; } // 写的是与主线程同一份物理页

int main(void) {
    pid_t pid = fork(); // 复制 mm_struct；父子 PTE 暂时同指一物理页且只读（COW）
    if (pid == 0) {
        g = 42; // 首次写入触发 page fault，内核复制页面并恢复可写
        printf("child  g=%d &g=%p\n", g, (void *)&g);
        _exit(0);
    }
    wait(NULL);
    // g 仍为 1：&g 的虚拟地址与子进程完全一致，但页表翻译到不同物理页
    printf("parent g=%d &g=%p\n", g, (void *)&g);

    pthread_t t;
    pthread_create(&t, NULL, thr, NULL); // 等价 clone(CLONE_VM|...|CLONE_THREAD)
    pthread_join(t, NULL);
    printf("after thread g=%d\n", g); // 42：无 COW，无页表切换，直接落到同一物理页
    return 0;
}

观察要点：父子进程打印出完全相同的 &g，这一句就直接否定了「进程间可以传指针」的任何可能性，也解释了为什么跨进程数据必须序列化而跨线程只需传地址。
```

### 4. 常见误区与进阶思考
误区 1：「线程一定比进程轻，所以并发一律上线程」。
(a) 创建开销被高估：Linux 的 fork 因 COW，与 clone 同量级（微秒级），真正昂贵的是切换时的 CR3 写入 + TLB 失效 + cache 冷启动，以及故障域代价。
(b) 无故障隔离：地址空间共享意味着任一野指针触发 SIGSEGV 会终止整个进程；而多进程模型下单个 worker 崩溃可被拉起，其余继续服务。
(c) 同步与 cache 成本：共享可变状态需要锁/原子/内存序；锁竞争与伪共享（false sharing，不同核写同一 cache line 上不同变量导致 line 在核间乒乓）在高并发下可能让吞吐低于多进程方案。
(d) 运行时层的线程未必轻：Node worker_threads 每个 worker 有独立 V8 isolate、独立 event loop、独立 GC，堆不共享（除 SAB），启动数十毫秒、内存数十 MB；Java 21 之前 1:1 映射 + 约 1MB 栈，正是这些成本催生了 virtual thread。

误区 2：「多进程不能共享内存 / 多线程共享一切」。两句都错。
- 多进程可以共享物理内存：mmap(MAP_SHARED)、POSIX shm_open、memfd_create 让不同进程的页表指向同一物理页，此时共享语义在缓存一致性层面与线程共享地址空间等价（同一物理地址，由 MESI 保证一致），差别仅在虚拟地址可能不同、且没有语言级对象模型。Node 的 SAB 也能通过 IPC 句柄传递给子进程实现跨进程共享内存。
- 多线程并不共享一切：每个线程有独立的用户栈、内核栈、寄存器组、TLS、errno、信号掩码、setjmp 上下文。把栈上局部变量地址交给其他线程使用即是悬垂指针。
- 附带纠正：教科书「进程是资源分配单位、线程是调度单位」是对概念的抽象，Linux 内核里不存在进程/线程类型之分，只有 task_struct 与 clone flags，「进程」是 tgid 相同的一组 task 的视图。

思考题：
父进程 fork 出子进程后，双方对同一个全局变量的写入互不可见；但双方若通过 mmap(MAP_SHARED) 映射同一文件，对同一字节的写入却互相可见。请依次从（1）MMU 页表翻译路径中 PTE 指向的物理页帧号是否相同、（2）COW 的写保护位与 page fault 处理流程、（3）多核 MESI 缓存一致性协议对同一物理地址的保证，这三个层次解释这两类行为的本质差异。并进一步回答：为什么 Node 的 worker_threads 不能像 C 线程那样直接共享普通堆对象，而必须退化为 SharedArrayBuffer + structured clone？（提示：从 V8 isolate 的堆是各自独立的地址区间、GC 需要精确的指针可达性与跨线程写屏障这两点切入。）
