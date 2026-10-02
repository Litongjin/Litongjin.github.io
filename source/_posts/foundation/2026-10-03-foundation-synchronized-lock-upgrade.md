---
title: "每日基础技术总结 · 2026-10-03 · synchronized 锁升级：偏向/轻量/重量锁"
date: 2026-10-03 07:01:54
categories: [技术分享]
tags: ["技术分享", "Java 后端与 Spring 生态"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-03 · synchronized 锁升级：偏向/轻量/重量锁

## 📚 今日主题

> **synchronized 锁升级：偏向/轻量/重量锁**（Java 后端与 Spring 生态）

### 1. 核心概念速览
synchronized 是 JVM 内建监视器锁，基于对象头（Mark Word）与 monitor 实现，用于保证临界区互斥与内存可见性。锁升级是 JVM 针对无竞争、低竞争、高竞争场景的渐进优化：偏向锁（无竞争）、轻量级锁（低竞争）、重量级锁（高竞争）。本质上它重新利用了对象头的位模式，避免或减少操作系统级互斥量（mutex）的系统调用，让锁成本与竞争程度匹配。该机制是理解 Java 并发、JVM 底层内存布局、以及未来转向后端高并发调优的基石；专业工程师必须掌握，否则无法诊断锁竞争、死锁、性能抖动等真实问题。

### 2. 底层原理剖析
每个 Java 对象的对象头含 Mark Word，JVM 用它的比特位编码不同锁状态。

1. 偏向锁：当首个线程进入同步块，JVM 将对象头中的偏向线程 ID 设为该线程，并记录 epoch。之后该线程再次进入同步块，只需比对 ID 即可，无需 CAS。若其他线程竞争，偏向锁被撤销（revoke），进入轻量级锁或直接升级。

2. 轻量级锁：线程在栈帧中创建锁记录（Lock Record），通过 CAS 将对象头 Mark Word 替换为指向锁记录的指针。成功则获得锁，失败则锁膨胀为重量级锁。解锁时反向 CAS 还原。

3. 重量级锁：依赖 ObjectMonitor，通过 OS 的 pthread mutex 与条件变量实现。未获得锁的线程阻塞并切换上下文，成本最高。

伪代码描述：

对象头状态流转：
无锁 001 -> 偏向锁 101 -> 轻量级锁 00 -> 重量级锁 10

加锁算法：
current = threadId()
if (obj.header.biasable && obj.header.biasedTo == null) {
    obj.header.biasedTo = current;  // 偏向
} else if (obj.header.biasedTo == current) {
    // 重入，直接进入
} else {
    if (lockRecord = newLockRecord()) {
        if (CAS(obj.header, biasedHeader, lockRecordPtr)) {
            // 获得轻量级锁
        } else {
            inflateToMonitor(obj);  // 重量级锁
        }
    }
}

与前端已有概念的对比：Java 的 synchronized 本质上更像是一门语言级别的互斥原语，类似于 JS 中的 `Atomics` 或 `SharedArrayBuffer` 的操作，但 synchronized 还内置了重入性、内存屏障和锁升级策略；而 JS 单线程事件循环天然规避了共享内存竞争，synchronized 与之不在同一抽象层。

前端工程师熟悉的 `async/await` 并发模型是非抢占式协作调度，而 synchronized 是抢占式互斥；理解后者需要回归到 CPU 原子指令（如 cmpxchg）与 OS 线程调度原语，而前端几乎不需要触达该层级。

### 3. 基础代码与实战验证
```text
以下极简代码验证锁升级的存在（JDK 15+ 需显式开启偏向锁，JDK 15 前默认启用）：

// 注：此处按你的要求不使用 Markdown 代码围栏，仅用缩进表示代码
public class LockUpgradeDemonstration {
    private static final Object LOCK = new Object();

    public static void main(String[] args) throws Exception {
        // 偏向锁阶段：单个线程获取锁后不会触发 CAS，直接惯用对象头
        synchronized (LOCK) {
            System.out.println("同一个线程重入，偏向锁无竞争");
        }

        // 使用 JOL（Java Object Layout）查看锁状态位
        // 偏向锁：0x0000000000000005 (101)
        // 轻量级锁：0x0000000000000000 (00)
        // 重量级锁：0x000000000000000A (10)
        // 执行时添加 -XX:BiasedLockingStartupDelay=0

        Thread competitor = new Thread(() -> {
            synchronized (LOCK) {
                System.out.println("第二个线程竞争，触发偏向锁撤销");
            }
        });
        competitor.start();
        competitor.join();

        // 此时锁可能升级为轻量级锁或重量级锁
        synchronized (LOCK) {
            Thread.sleep(1000);  // 若 JVM 已膨胀，保持重量级
        }
    }
}

核心注释：
- synchronized 编译为 monitorenter/monitorexit 字节码，JVM 运行时解释为完整的升级路径。
- 偏向延迟设为 0 才能观察到偏向锁，否则 JVM 在启动后的 4 秒内强制无锁状态。
- 竞争线程的加入会让 JVM 调用 revoke_bias，暂停安全点（SafePoint）执行，开销较大，这解释了为什么偏向锁在高竞争下反而拖累性能。

由于锁状态位在对象头中不可直接读，建议使用 JOL 工具输出 Mark Word 十六进制验证，代码此处不展开。
```

### 4. 常见误区与进阶思考
误区 1：认为『锁只能从偏向到轻量再到重量，逐级单向升级』。实际上 JVM 会在批量重偏向、批量撤销等场景中动态调整偏向锁行为；且轻量级锁在竞争极低时不会立刻升级为重量级锁，而是可能用自旋优化。锁升级是启发式路径，不是严格的状态机。

误区 2：认为 synchronized 一定比 Lock 接口慢。重量级锁经过 JVM 的锁粗化、锁消除、自适应自旋优化后，在低竞争场景下性能与 Lock 相当；偏向锁在 JDK 15 中默认废弃移除，证明了过时的锁优化会成为一种反模式。

思考题：当两个线程交替执行但每次只单独持锁，锁状态该如何演进？偏向锁的 epoch 机制如何避免每次撤销都触发一次 Stop-The-World？试从 JVM 源码 `biasedLocking.cpp` 的角度推导其流程，并回答为什么高并发下偏向锁违反 Ahmdal 定律的收益预期。
