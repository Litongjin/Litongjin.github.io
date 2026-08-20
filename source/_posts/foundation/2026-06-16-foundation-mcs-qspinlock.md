---
title: "每日基础技术总结 · 2026-06-16 · 自旋锁与锁竞争：MCS 锁与 qspinlock"
date: 2026-06-16 08:00:00
categories: [技术分享]
tags: ["技术分享", "操作系统基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-06-16 · 自旋锁与锁竞争：MCS 锁与 qspinlock

## 📚 今日主题

> **自旋锁与锁竞争：MCS 锁与 qspinlock**（操作系统基础）

### 1. 核心概念速览
自旋锁（spinlock）是一种忙等待互斥原语：线程在获取锁失败时持续循环检测锁状态，而非睡眠。锁竞争指多个线程同时尝试获取同一把锁，导致争抢的物理核心在缓存一致性协议（如MESI）下反复无效化cacheline，产生严重的总线流量和内存屏障开销。MCS锁（Mellor-Crummey and Scott）通过为每个等待线程分配一个显式队列节点，使每个线程自旋在本地节点上，避免全局共享变量的竞争，将锁获取复杂度降为O(1)且无全局cacheline抖动。qspinlock（queued spinlock）是Linux内核采用的融合方案：低并发时使用简化的TAS（test-and-set）自旋，高竞争时自动升级为MCS排队机制，同时通过按字节编码的尾指针在32位中实现队列节点复用。该机制处于操作系统内核并发控制的核心位置，是理解多核性能、锁竞争、cacheline亲和性以及无锁数据结构的基础。专业工程师必须掌握，因为所有高并发系统的性能瓶颈本质上是锁竞争导致的缓存一致性开销，而非锁本身的算法复杂度。

### 2. 底层原理剖析
传统自旋锁（如Linux早期raw_spinlock）本质是一个原子变量，通过原子操作（如xchg或cmpxchg）置位。获取锁时原子交换，若锁已被占用则循环读取该变量。问题在于：所有等待线程共享同一cacheline，每次持锁线程释放时，锁变量被写，导致所有等待者的cacheline失效，然后所有线程同时原子操作，只有一个成功，其余继续失效——即“惊群”与cacheline颠簸。MCS锁的核心是队列：每个线程在栈上或结构体中分配一个mcs_node，包含next指针和locked标志。锁本身指向队尾节点。获取锁时：1）原子交换锁指针，将本节点设为队尾，并得到前驱节点；2）若前驱为NULL，则直接获得锁；3）否则将前驱的next指向本节点，然后自旋在本节点的locked字段上。释放锁时：1）若本节点的next为NULL，则原子比较交换锁指针，若成功则释放完成；2）否则将下一个节点的locked置为1，由下一个节点解除自旋。这样每个自旋者只在自己的节点上自旋，该节点由前驱写入一次，避免多线程共享cacheline。qspinlock在MCS基础上优化：锁用32位原子变量，低8位为lock字节，剩余24位为尾指针索引。每个CPU有一个预分配的MCS节点数组。当只有一个线程等待时，直接用lock字节TAS获取；当竞争加剧，尾指针编码队列头尾，进入排队。释放时若队列存在，则直接解锁并唤醒下一个，无需在lock字节上做无意义竞争。与前端概念对比：Java接口是运行时多态（JVM方法分派），TypeScript接口是编译期结构类型（静态约束）；两者都定义契约，但一个作用于运行时的对象分派，一个作用于编译时的类型检查。类似地，传统自旋锁和MCS锁都提供互斥契约，但传统自旋锁在锁变量上全局自旋（类似运行时的动态竞争），而MCS锁在编译期/结构上预先分配队列节点（类似静态类型约束），将竞争分散到局部状态，本质上是把全局共享的写冲突转化为局部写，类似TS的“结构化类型”将Java的“名义类型”的运行时检查提前到编译期。

### 3. 基础代码与实战验证
```text
以下为MCS锁的C语言实现核心逻辑（省略内存屏障细节）：

struct mcs_node {
    struct mcs_node *next;
    int locked; // 0表示未获得，1表示已获得
};

struct mcs_lock {
    struct mcs_node *tail; // 指向队尾节点，初始NULL
};

// 获取锁：每个线程传入自己的本地节点node
void mcs_acquire(struct mcs_lock *lock, struct mcs_node *node) {
    struct mcs_node *pred;
    node->next = NULL;
    // 原子交换：将lock->tail设为node，并返回旧tail作为前驱
    pred = __atomic_exchange_n(&lock->tail, node, __ATOMIC_ACQUIRE);
    if (pred == NULL) {
        // 前驱为空，说明无人持有，当前线程直接获得锁
        return;
    }
    // 让前驱知道自己是后继者
    __atomic_store_n(&pred->next, node, __ATOMIC_RELEASE);
    // 自旋等待前驱将node->locked置1
    while (!__atomic_load_n(&node->locked, __ATOMIC_ACQUIRE)) {
        // 仅自旋在自己的cacheline上，不会造成全局缓存竞争
    }
}

void mcs_release(struct mcs_lock *lock, struct mcs_node *node) {
    struct mcs_node *next;
    next = __atomic_load_n(&node->next, __ATOMIC_ACQUIRE);
    if (next == NULL) {
        // 当前无后继，尝试把lock->tail从node置为NULL
        // 若成功，说明没有新等待者，锁完全释放
        if (__atomic_compare_exchange_n(&lock->tail, &node, NULL,
                                        false, __ATOMIC_RELEASE, __ATOMIC_RELAXED)) {
            return;
        }
        // 否则说明有新的等待者插入了队列，此时需要等待其next被设置
        while ((next = __atomic_load_n(&node->next, __ATOMIC_ACQUIRE)) == NULL) {
            // 空转等待，CPU需执行暂停指令以降低功耗
        }
    }
    // 将后继节点的locked置1，使其获得锁并停止自旋
    __atomic_store_n(&next->locked, 1, __ATOMIC_RELEASE);
}

// 验证方式：两个线程分别传入各自的节点node0和node1，调用acquire/release，观察是否互斥。
// 关键点：每个线程自旋在node->locked，该字段只有前驱线程写入，不会与其他线程共享。
// qspinlock的实现类似，但将tail指针和locked标志压缩在一个32位整数中，且每个CPU有固定节点数组。
```

### 4. 常见误区与进阶思考
误区1：认为自旋锁适合所有短临界区。实际上，即使临界区极短，如果锁竞争激烈且线程数多于CPU数，忙等待会浪费大量CPU周期；更致命的是，MCS锁虽然消除了cacheline竞争，但每个等待线程自旋仍会占用CPU流水线，因此在内核中qspinlock还会结合抢占、调度和睡眠锁（如mutex）来避免过度自旋。
误区2：忽略内存序和编译屏障。自旋锁的正确性依赖原子操作的内存序语义（acquire/release），若只使用普通赋值和循环，即使在x86上可能工作，在ARM等弱内存模型上会崩溃。前端开发者常忽略异步竞态中的时序，类似地，锁的释放必须用release语义，获取必须用acquire语义。
思考题：在MCS锁释放时，如果当前节点的next为NULL，并且CAS将tail置NULL失败（说明有新的等待者插入），为何必须等待next被赋值？如果此时直接返回，会发生什么？请从队列一致性和内存序角度分析。
