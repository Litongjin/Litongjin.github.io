---
title: "每日基础技术总结 · 2024-01-08 · ConcurrentHashMap：分段锁到 CAS+synchronized"
date: 2024-01-08 20:00:00
categories: [技术分享]
tags: ["技术分享", "Java 后端与 Spring 生态"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-01-08 · ConcurrentHashMap：分段锁到 CAS+synchronized

## 📚 今日主题

> **ConcurrentHashMap：分段锁到 CAS+synchronized**（Java 后端与 Spring 生态）

### 1. 核心概念速览
ConcurrentHashMap (CHM) 是 Java 标准库中为高并发读写场景设计的高性能线程安全哈希表。其核心解决的问题是消除传统 HashTable（全局锁）和 Collections.synchronizedMap（粗粒度锁）带来的写-读、写-写阻塞瓶颈，同时避免 java.util.HashMap 在多线程下的数据竞争（Data Race）导致的无限循环或数据丢失。

机制演进：Java 1.7 采用 Segment（分段锁）架构，将 Map 划分为 N 个 Segment，每个 Segment 继承自 ReentrantLock，仅锁定当前桶链表/树；Java 1.8+ 摒弃 Segment，转为 CAS + synchronized 细粒度锁，以 Bucket（数组节点）为单位进行同步，支持红黑树优化长链查找。在计算机体系结构中，它体现了从‘进程/线程资源隔离’到‘内存级原子操作优化’的演进，是理解并发控制、内存可见性（volatile）、CPU Cache Line 伪共享及无锁编程思想的关键基石。专业工程师必须掌握它以正确评估系统吞吐量瓶颈及避免死锁隐患。

### 2. 底层原理剖析
底层运行机制剖析（基于 Java 8+）：
1. 数据结构：Node[K,V] 数组 + 链表 + 红黑树（当链表长度 > TREEIFY_THRESHOLD=8 且数组容量 >= MIN_TREEIFY_CAPACITY=64 时转换）。所有 Node 的 key/value 均为 final，确保不可变性，配合 volatile 引用实现读取可见性。
2. 写入流程（putVal）：
   a. 计算 Hash：通过扰动函数防止低位冲突，保证散列均匀。
   b. CAS 插入首节点：若目标桶为空，使用 Unsafe.compareAndSwapObject 尝试将新节点设为头节点。成功则直接返回，无需加锁（无竞争状态）。
   c. FOWARDING_NODE 处理：若检测到 MOVED 标志位（扩容中），协助其他线程转移数据。
   d. Synchronized 锁桶：若桶非空且无移位标志，对桶头节点 synchronized 加锁。注意：synchronized 作用于 Node 实例而非 Class，且锁范围极小（仅当前桶）。
   e. 插入或更新：在锁内判断键是否已存在，存在则替换 value，不存在则追加至链表尾部或平衡红黑树。
3. 读取流程（get）：完全无锁。利用 volatile 语义保证读到最新的 Node 引用，再读取 final 字段值。依赖 CPU 硬件层面的 Store Buffer 和 Memory Order Barrier 保证顺序性。
4. 扩容机制：多线程协作扩容。检测阈值触发后，创建更大数组，迁移元素。迁移时使用 ForwardingNode（占位符）引导线程参与重哈希，避免单点瓶颈。

对比前端概念：Java 的 ConcurrentHashMap 类似于 TypeScript 中结合 Immutable.js 与 Web Worker 的思路，但更底层。TS 接口仅定义类型契约，不保证运行时线程安全；而 CHM 在运行时通过 CAS 指令级原子性和 JVM 内存模型（JMM）保障一致性。前端 EventLoop 是基于事件驱动的异步单线程模型，不存在真正的内存级数据竞争；CHM 则是针对多核 CPU 多线程共享内存空间的互斥与原子性解决方案，本质区别在于‘时间片轮转’与‘物理并行’的差异。

### 3. 基础代码与实战验证
```text
// Java 8+ ConcurrentHashMap 核心行为演示
import java.util.concurrent.ConcurrentHashMap;

public class CHMDemo {
    public static void main(String[] args) {
        // 1. 初始化：默认并发度 16，负载因子 0.75
        ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

        // 2. putIfAbsent: 原子性检查与插入
        // 底层逻辑：先 CAS 获取旧值，若 null 则 CAS 插入新值；若并发冲突，重试或加锁
        Integer result = map.putIfAbsent("key", 1);
        if (result == null) {
            // 执行路径 A: 当前线程独占获得该桶访问权或桶为空，原子写入成功
            System.out.println("Insert succeeded");
        } else {
            // 执行路径 B: 键已存在，返回现有值，未修改
            System.out.println("Already exists: " + result);
        }

        // 3. computeIfAbsent: 细粒度锁验证
        // 关键点：lambda 仅在 Key 不存在时执行，且整个 Lambda 执行过程被 synchronized 保护
        // 避免重复计算和竞态条件
        int value = map.computeIfAbsent("count", k -> {
            // 模拟耗时操作
            try { Thread.sleep(10); } catch (InterruptedException e) {}
            return 100;
        });

        // 4. get: 无锁读取
        // 底层：volatile read 最新 Node 引用 -> Node.value(final)
        // 可能读到稍旧的数据（弱一致性），但绝不会读到脏数据或部分构造对象
        System.out.println("Current Value: " + map.get("count"));

        // 5. size(): 近似计数
        // 底层：累加 CounterCell 数组的高阶位与低阶位，结合 baseCount
        // 允许一定误差，O(1) 复杂度，避免遍历全表
        System.out.println("Approx Size: " + map.size());
    }
}
```

### 4. 常见误区与进阶思考
常见误区：
1. ‘绝对实时一致性’误解：认为 CHM 的 get() 能立刻看到 put() 的结果。实际上，由于 volatile 的可见性延迟和 CPU 缓存一致性协议（MESI），在高并发下可能存在微秒级的滞后。对于需要强一致性的业务（如金融交易校验），CHM 不适用，应使用 ReadWriteLock 或数据库事务。
2. ‘迭代器安全性’误用：THM 的迭代器是 Fail-Fast 但弱一致的，遍历时不能依靠抛出 ConcurrentModificationException 来判断数据变更（因为设计上就不抛异常阻止遍历，而是反映快照期间的状态）。若在遍历期间依赖数据完整性做业务决策，会导致逻辑错误。

进阶思考题：
在 Java 8 中，为何选择 synchronized 代替 ReentrantLock 来实现细粒度锁？请结合 JVM Hotspot 的轻量级锁升级机制（偏向锁->轻量级锁->重量级锁）以及 CAS 操作的开销，分析这种设计在 JEP 102（改进 AUIRC 和 Reduce Contention in ConcurrentHashMap）之后的性能权衡依据。
