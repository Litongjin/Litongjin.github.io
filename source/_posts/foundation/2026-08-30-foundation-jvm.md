---
title: "每日基础技术总结 · 2026-08-30 · JVM 对象头与锁升级"
date: 2026-08-30 08:00:00
categories: [技术分享]
tags: ["技术分享", "编程语言底层"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-30 · JVM 对象头与锁升级

## 📚 今日主题

> **JVM 对象头与锁升级**（编程语言底层）

### 1. 核心概念速览
JVM对象头是堆中对象实例前部的元数据区域，包含Mark Word（标记字）、Klass Pointer（类型指针）和数组长度（若为数组）。Mark Word通过动态复用位区间，承载对象哈希码、GC分代年龄、锁状态标记（无锁、偏向锁、轻量级锁、重量级锁）以及对应的锁记录指针/监视器指针。锁升级是synchronized在对象监视器上从无锁到偏向锁、轻量级锁（CAS自旋）、重量级锁（OS Mutex/ObjectMonitor）的逐步膨胀机制，本质是在并发竞争强度与上下文切换代价之间做适应式权衡。它解决的是：在多线程临界区竞争程度未知的情况下，避免锁实现一刀切带来的性能浪费。该机制位于JVM运行时数据布局与执行引擎层面，是理解Java并发、性能调优、JOL（Java Object Layout）分析的知识基座；专业工程师必须掌握它，才能解释synchronized的实际开销、死锁诊断中的monitor信息、以及高并发场景下锁竞争优化的底层依据。在更宏观的体系中，它处于操作系统线程调度与高级语言并发原语之间的适配层；对使用JVM系技术栈的AI推理/服务端工程而言，锁竞争直接影响吞吐上限与延迟分位数，因此必须理解其底层逻辑。

### 2. 底层原理剖析
以64位HotSpot JVM为例，对象头典型布局为：Mark Word占64bit，Klass Pointer在开启压缩指针时占32bit（否则64bit），数组对象额外有32bit记录长度。Mark Word的锁标志位（低位2bit）决定锁状态：01表示无锁或偏向，00表示轻量级锁，10表示重量级锁，11表示GC标记。若偏向锁可用，还会用第3bit（biased_lock）区分无锁(0)和偏向锁(1)。

锁升级的核心流程：
1. 无锁：Mark Word保存identity hashcode、GC分代年龄、biased_lock=0。
2. 偏向锁：第一个线程进入synchronized块时，JVM通过CAS将Mark Word的偏向线程ID（54bit）、epoch、biased_lock=1写入。此后同一线程重入该临界区时，只需检查线程ID匹配即可，不再执行CAS，开销仅为一次内存读比较。
3. 轻量级锁：另一个线程尝试获取该锁时，偏向锁需要撤销（revoke）。撤销在安全点进行：若原偏向线程仍存活且在临界区，则升级为轻量级锁。竞争线程在当前线程栈帧中创建Lock Record，拷走Mark Word（Displaced Mark Word），并通过CAS将对象头Mark Word指向Lock Record，锁标志置00。后续线程通过自旋等待CAS成功。
4. 重量级锁：若自旋超过阈值仍失败，则锁膨胀为重量级锁。Mark Word指向ObjectMonitor，线程挂起到内核态，通过操作系统互斥量/条件变量阻塞唤醒。

轻量级锁的解锁同样关键：线程将Lock Record中保存的Displaced Mark Word通过CAS交换回对象头；若成功说明无竞争，若失败说明已有其他线程修改过Mark Word（即发生竞争），此时需要膨胀为重量级锁。锁升级过程不是单调“越变越重”后才降级；轻量级锁释放后对象可回到无锁状态，但一旦膨胀为重量级锁，在JVM中不可直接降级（需依赖GC或重启）。

与前端已有知识的对比：TypeScript的interface只是编译期类型约束，编译后无任何运行时内存结构；而Java对象头是运行时真实存在的内存布局元数据。JavaScript V8引擎中的HiddenClass/Map和PropertyDescriptor概念上更像JVM的Klass Pointer和字段布局，但JS没有用户态锁结构，也不存在锁升级。前端并发模型（单线程事件循环）天然规避了锁竞争；只有Web Worker+SharedArrayBuffer+Atomics的场景才类似低层并发原语，但也没有JVM这种基于对象头的偏向锁/自旋升级机制。

### 3. 基础代码与实战验证
```text
以下代码使用JOL（Java Object Layout）打印对象头布局，观察无锁、加锁、调用hashCode后的Mark Word状态变化。需要引入依赖：org.openjdk.jol:jol-core。运行JVM参数建议：-XX:+UseBiasedLocking -XX:BiasedLockingStartupDelay=0（JDK 15后需显式开启偏向锁，且需使用JDK 11或更早版本验证完整效果）。

import org.openjdk.jol.info.ClassLayout;

public class LockUpgradeDemo {

    static final Object lock = new Object();

    public static void main(String[] args) throws InterruptedException {
        // 对象刚创建时：Mark Word为无锁状态，biased_lock=0，锁标志位为01
        System.out.println("无锁状态:\\n" + ClassLayout.parseInstance(lock).toPrintable());

        // 进入同步块：monitorenter/monitorexit 字节码指令
        // 若开启偏向锁且未调用hashCode，当前线程ID会被CAS写入Mark Word
        // 同一线程重入时，仅比较标记线程ID，不再做CAS
        synchronized (lock) {
            System.out.println("同步块内:\\n" + ClassLayout.parseInstance(lock).toPrintable());
        }

        // 调用hashCode：identity hashcode需要写入Mark Word
        // 但Mark Word中偏向线程ID占用了空间，所以必须撤销偏向锁才能生成hashCode
        int h = lock.hashCode();
        System.out.println("hashCode: " + h);
        System.out.println("调用hashCode后:\\n" + ClassLayout.parseInstance(lock).toPrintable());

        // 观察输出中Mark Word的二进制/十六进制和锁标志位，即可验证以下结论：
        // 1. 无锁时mark word低位为01，且hashCode区在调用前为0；调用后填入identity hashcode。
        // 2. 同步块内如果仍是偏向锁，mark word中会显示可能的线程ID；若发生竞争则可能升级为轻量级锁（低位00）。
        // 3. 调用hashCode后，即使对象不再被偏向，Mark Word中hashCode区已有值，之后该对象无法再进入偏向锁状态。
    }
}
```

### 4. 常见误区与进阶思考
误区1：认为synchronized一开始就是重量级锁。实际上JVM在开启偏向锁时，无竞争场景下会先使用偏向锁，甚至不经过CAS重入；发生轻度竞争时才升级为轻量级锁，依赖自旋；只有自旋失败才膨胀到重量级锁。表现为锁升级路径是逐步的，而非一步到位。

误区2：认为锁一旦升级就永远保持高级锁。轻量级锁在释放时如果CAS成功，对象头会恢复为无锁状态；只有已经膨胀为重量级锁后才不可直接“降级”。另外，偏向锁的撤销本身有安全点停顿成本，这是偏向锁在JDK 15之后被默认禁用（deprecated/disabled）的原因之一。

进阶思考题：在开启偏向锁（-XX:+UseBiasedLocking -XX:BiasedLockingStartupDelay=0）的环境中，一个线程反复进入同一个对象的synchronized块，且没有其他线程竞争。此时锁状态会经历什么？如果该线程在第一次进入同步块之前调用了该对象的hashCode()，结果又会怎样？请从Mark Word的位空间和锁定代价的角度解释。
