---
title: "每日基础技术总结 · 2026-06-16 · 条件变量与互斥锁的底层实现：futex"
date: 2026-06-16 08:00:00
categories: [技术分享]
tags: ["技术分享", "操作系统基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-06-16 · 条件变量与互斥锁的底层实现：futex

## 📚 今日主题

> **条件变量与互斥锁的底层实现：futex**（操作系统基础）

### 1. 核心概念速览
futex（Fast Userspace Mutex）是 Linux 内核提供的同步原语，本质是“用户态原子操作 + 内核态等待队列”的组合。它解决的核心问题是：当多线程竞争一个资源时，如何在绝大多数无竞争场景下不陷入内核，仅在真正需要阻塞/唤醒时才进行系统调用。机制：线程先通过用户态原子指令（如 cmpxchg）尝试修改一个共享整数；失败后调用 futex_wait(addr, val) 系统调用，内核检查 *addr == val，若成立则挂起该线程并加入该地址对应的等待队列；解锁方通过 futex_wake(addr, n) 唤醒队列上 n 个线程。futex 位于操作系统并发控制的中枢位置，是 pthread 互斥锁、条件变量、信号量等用户态同步库的底层支撑。专业工程师必须掌握它，因为所有后端高并发系统的性能、死锁、优先级反转问题最终都要归结到锁的底层代价与行为；不理解 futex，就无法精确分析锁竞争导致的上下文切换开销、无法理解条件变量为何必须与互斥锁配合、也无法调试神秘的“丢失唤醒”类问题。

### 2. 底层原理剖析
底层运行机制如下：
互斥锁（mutex）的核心是共享状态 state，取值 0（未锁）、1（已锁无等待者）、2（已锁有等待者）。加锁时，先执行用户态原子交换 `old = atomic_exchange(&state, 1)`。若 old == 0，加锁成功，全程未进入内核。若失败，说明锁被占用，线程进入慢路径：将 state 从 1 升级为 2（表示有等待者），然后调用 futex_wait(&state, 2)。内核比较 *state 是否等于 2，若等于则挂起当前线程；若不等于（说明锁可能已被释放）则立即返回，线程重新尝试加锁。解锁时，执行 `old = atomic_exchange(&state, 0)`；若 old == 1，说明无等待者，直接返回；若 old == 2，说明有等待者，调用 futex_wake(&state, 1) 唤醒一个线程。
条件变量（condvar）解决的是“等待某个条件成立”的阻塞。它内部维护一个序列号 count。cond_wait(cv, mutex) 必须在持有 mutex 时调用：先记录当前 count 值，然后释放 mutex（确保其他线程可进入临界区），再调用 futex_wait(&cv->count, seq)。当 count 被 signal 修改并 futex_wake 后，wait 返回，线程重新获取 mutex，并检查条件是否满足（因虚假唤醒需循环）。cond_signal(cv) 在持有 mutex 时执行：原子增加 count 并调用 futex_wake。这里的序列号机制保证了“释放锁”和“进入睡眠”之间的原子性，避免信号丢失——因为 futex_wait 会校验 count 值，若 signal 已经增加过 count，则 wait 不会真正睡眠。
与前端已有概念的对比：前端 JavaScript 是单线程事件循环模型，不存在用户态线程抢占，异步通过任务队列实现；而 futex 是抢占式多线程操作系统下的阻塞原语，允许线程主动挂起并让出 CPU。正如 Java 接口与 TypeScript 接口虽然都叫“接口”，但语义和运行期行为完全不同——Java 接口是运行期类型约束，TS 接口仅存在于编译期；同样，前端开发者不能因为 Promise 的 then 链看起来像“锁”，就将语言层面的调度机制与内核 futex 混淆。futex 强调的是“用户态优先，内核兜底”的设计哲学，与事件循环的“永不阻塞”哲学有本质差异。

### 3. 基础代码与实战验证
```text
// 基于 futex 的互斥锁（Linux，极简实现）
#include <linux/futex.h>
#include <sys/syscall.h>
#include <unistd.h>
#include <stdatomic.h>

typedef struct { atomic_int state; } futex_mutex;
// state: 0=未锁, 1=已锁且无等待者, 2=已锁且有等待者

static long futex_wait(int *uaddr, int expect) {
    return syscall(SYS_futex, uaddr, FUTEX_WAIT, expect, NULL, NULL, 0);
}
static long futex_wake(int *uaddr, int n) {
    return syscall(SYS_futex, uaddr, FUTEX_WAKE, n, NULL, NULL, 0);
}

void mutex_lock(futex_mutex *m) {
    // 快速路径：原子交换，将 state 置1；若原值为0则直接获得锁，无系统调用
    int prev = atomic_exchange_explicit(&m->state, 1, memory_order_acquire);
    if (prev == 0) return;

    while (1) {
        // 慢路径：确保 state 为2（有等待者），然后调用 futex_wait
        // 若 prev==2 或交换后返回值非0，说明锁仍被占用，需要睡眠
        if (prev == 2 || atomic_exchange_explicit(&m->state, 2, memory_order_acquire) != 0) {
            // 内核检查 *state == 2 才挂起；否则立即返回，避免丢失唤醒
            futex_wait((int*)&m->state, 2);
        }
        // 醒来后重新尝试获取锁
        prev = atomic_exchange_explicit(&m->state, 1, memory_order_acquire);
        if (prev == 0) return;
    }
}

void mutex_unlock(futex_mutex *m) {
    // 原子置0；若原值为2，说明存在等待线程，需要唤醒一个
    int prev = atomic_exchange_explicit(&m->state, 0, memory_order_release);
    if (prev == 2) futex_wake((int*)&m->state, 1);
}

// 条件变量（简化）：使用序列号 + futex，与互斥锁配合
typedef struct { atomic_int count; } futex_cond;

void cond_wait(futex_cond *c, futex_mutex *m) {
    // 必须在持有 m 时调用；记录当前序列号
    int seq = atomic_load_explicit(&c->count, memory_order_acquire);
    mutex_unlock(m);                    // 释放互斥锁，让其他线程进入临界区
    // 仅当 count 仍等于 seq 时睡眠；若 signal 已执行，则立即返回
    futex_wait((int*)&c->count, seq);
    mutex_lock(m);                      // 重新获取互斥锁（调用方需循环检查条件）
}

void cond_signal(futex_cond *c) {
    // 增加序列号并唤醒一个等待者
    atomic_fetch_add_explicit(&c->count, 1, memory_order_release);
    futex_wake((int*)&c->count, 1);
}
```

### 4. 常见误区与进阶思考
常见误区：
1. 将 futex 当作“锁”本身。futex 只是“快速用户态互斥”的底层原语，提供的是原子检查和阻塞/唤醒的机制；真正的锁语义（如重入、公平性）由用户态库（glibc）在之上构建。直接使用 futex 不会自动获得健壮的锁。
2. 认为 futex_wait 会无条件阻塞。实际上它携带期望值参数，内核会先比较 *uaddr 与期望值，不相等则立即返回。这一机制是为了避免“先睡眠再检查”导致的丢失唤醒。如果忽略这一点，写出不循环的等待代码，就会在极端竞争下永久阻塞。

思考题：若在 cond_wait 实现中，将顺序改为“先 futex_wait 再释放互斥锁”，会发生什么？请从锁持有和唤醒丢失两个角度分析。
