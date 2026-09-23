---
title: "每日基础技术总结 · 2026-09-24 · 死锁的四个必要条件与循环等待检测"
date: 2026-09-24 07:04:19
categories: [技术分享]
tags: ["技术分享", "操作系统基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-24 · 死锁的四个必要条件与循环等待检测

## 📚 今日主题

> **死锁的四个必要条件与循环等待检测**（操作系统基础）

### 1. 核心概念速览
死锁是指多个进程在执行过程中，因争夺资源而造成的一种互相等待的现象，若无外力作用，它们都将无法推进。其本质是资源分配图中存在环路（Cycle），且系统缺乏互斥、保持并等待、非抢占和循环等待这四个必要条件同时成立时的全局协调机制。在分布式系统与后端高并发场景中，死锁导致线程/进程无限期阻塞，引发服务不可用；掌握此概念是设计高可用并发模型、排查生产环境hang住问题的基石。

### 2. 底层原理剖析
死锁发生的四个必要条件是：1. 互斥条件（Mutual Exclusion）：资源一次只能被一个进程占用，体现资源的原子性与排他性；2. 保持并等待（Hold and Wait）：进程已持有至少一个资源，但又在请求新的被其他进程占有的资源；3. 非抢占条件（No Preemption）：已获得的资源在未使用完前不能被强行剥夺；4. 循环等待（Circular Wait）：若干进程之间形成一种头尾相接的循环等待资源关系。检测机制依赖于构建有向图（Resource Allocation Graph, RAG）。若图中无回路则系统中不可能发生死锁；若有回路且单实例资源则必然死锁；多实例资源需通过银行家算法等检测死结。与前端相比，TS接口用于静态类型约束编译期行为，而死锁是运行时空闲态的资源拓扑状态，前者消除类型错误，后者消除运行时依赖环。

### 3. 基础代码与实战验证
```text
// Java 示例：演示循环等待导致的死锁
// T1 持有 lockA 尝试获取 lockB
// T2 持有 lockB 尝试获取 lockA
public class DeadlockDemo {
    private static final Object lockA = new Object();
    private static final Object lockB = new Object();

    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            synchronized (lockA) { // 步骤1: T1 获取 lockA 所有权
                try { Thread.sleep(50); } catch (InterruptedException e) {}
                synchronized (lockB) { // 步骤2: T1 试图获取 lockB，进入 WAITING 状态
                    System.out.println("T1 done");
                }
            }
        }, "Thread-1");

        Thread t2 = new Thread(() -> {
            synchronized (lockB) { // 步骤3: T2 获取 lockB 所有权
                try { Thread.sleep(50); } catch (InterruptedException e) {}
                synchronized (lockA) { // 步骤4: T2 试图获取 lockA，进入 WAITING 状态
                    System.out.println("T2 done");
                }
            }
        }, "Thread-2");

        t1.start(); t2.start();
        // 底层监控：Object.wait() 内部维护了 CLH 队列，两个线程相互挂起，GC Roots 可达但不 runnable
    }
}
```

### 4. 常见误区与进阶思考
误区一：认为只要设置了超时机制（如 ReentrantLock.lockInterruptibly）就能彻底解决死锁。实际上，超时只是避免了永久阻塞，并未解决资源竞争的逻辑冲突，可能导致业务逻辑在半完成状态下回滚不一致。误区二：混淆互斥锁与读写锁的死锁特征。读写锁可能在写操作饥饿中表现类似死锁，但本质是调度策略而非拓扑环。进阶思考：在微服务架构中，如果两个服务的调用链互为依赖（Service A 同步调用 Service B，Service B 同步调用 Service A），这是否构成操作系统层面的死锁？请从 TCP 连接池满、线程池耗尽及协议重试风暴的角度分析其差异。
