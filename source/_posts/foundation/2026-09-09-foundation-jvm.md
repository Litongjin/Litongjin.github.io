---
title: "每日基础技术总结 · 2026-09-09 · JVM 对象头与锁升级"
date: 2026-09-09 08:00:00
categories: [技术分享]
tags: ["技术分享", "编程语言底层"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-09 · JVM 对象头与锁升级

## 📚 今日主题

> **JVM 对象头与锁升级**（编程语言底层）

### 1. 核心概念速览
JVM对象头是Java堆中每个对象实例的头部元数据，由Mark Word与Klass Pointer组成。Mark Word存储对象的hashCode、GC分代年龄、锁状态标志及锁相关指针，是JVM实现synchronized同步与GC的基础。锁升级是HotSpot VM针对synchronized的自适应优化：根据共享对象竞争激烈程度，动态将锁状态从无锁/偏向锁升级为轻量级锁、重量级锁，以最小化同步开销。其本质是一个基于CAS和操作系统线程调度的状态机，位于JVM内存布局与并发控制层面。专业工程师必须掌握它，因为对象头与锁升级直接决定高并发场景下锁的性能与行为，也是分析死锁、锁竞争、内存占用和性能调优的根基。

### 2. 底层原理剖析
64位HotSpot的Mark Word中，锁标志位在低2位：01表示无锁或偏向锁（倒数第3位为偏向位），00表示轻量级锁，10表示重量级锁，11表示GC标记。无锁时高54位存储hashCode；偏向锁时存储偏向线程ID及epoch。轻量级锁时Mark Word指向线程栈中的Lock Record；重量级锁时指向ObjectMonitor对象。

升级路径（单向、不可逆）：
1. 无锁→偏向锁：首个线程进入同步块时，JVM通过CAS将Mark Word中的偏向线程ID设为当前线程并置偏向位为1。此后该线程再次进入无需任何同步操作。
2. 偏向锁→轻量级锁：另一个线程尝试获取时，JVM需在全局安全点撤销偏向，将Mark Word复制到新线程的Lock Record，并通过CAS原子替换Mark Word为指向该记录，标志位变为00。若CAS失败（多线程竞争），则继续膨胀。
3. 轻量级锁→重量级锁：自旋等待失败或持锁时间过长，JVM将Mark Word指向ObjectMonitor，标志位变为10。后续线程进入Monitor的EntryList阻塞，由操作系统调度。
关键机制包括CAS、安全点、自旋（自适应自旋）和锁记录。

与前端V8引擎对比：V8中JavaScript对象在属性结构变化时，会从HiddenClass快速模式退化为字典模式，也是一种运行时根据访问模式动态调整内部表示的结构优化。JVM对象头与锁升级同样属于运行时自适应的元数据调整，但前者用于并发同步，后者用于属性访问优化。

### 3. 基础代码与实战验证
```text
// 极简验证代码（需配合JOL工具，JDK8环境，JVM参数：-XX:+UseBiasedLocking -XX:BiasedLockingStartupDelay=0）
import org.openjdk.jol.info.ClassLayout;

public class LockUpgradeCheck {
    static final Object lock = new Object(); // 对象头初始为无锁可偏向状态，mark word后两位=01，偏向位=0

    public static void main(String[] args) throws Exception {
        System.out.println("无锁：" + ClassLayout.parseInstance(lock).toPrintable());

        Thread t1 = new Thread(() -> {
            synchronized (lock) { // 仅一个线程进入 → 偏向锁，mark word高54位存线程ID，偏向位=1
                System.out.println("偏向锁：" + ClassLayout.parseInstance(lock).toPrintable());
            }
        });
        t1.start(); t1.join();

        Thread t2 = new Thread(() -> {
            synchronized (lock) { // 第二个线程开始竞争 → 撤销偏向，升级为轻量级锁，mark word指向Lock Record，后两位=00
                System.out.println("轻量级锁：" + ClassLayout.parseInstance(lock).toPrintable());
            }
        });
        t2.start(); t2.join();

        // 若启动多个线程同时进入临界区，可观察到mark word指向ObjectMonitor，后两位=10，即重量级锁
        System.out.println("验证完成");
    }
}
```

### 4. 常见误区与进阶思考
误区1：认为锁升级必然经历“无锁→偏向→轻量→重量”全链路。实际偏向锁可能因批量撤销而直接回到无锁，或新线程通过CAS直接获取轻量级锁；轻量级锁自旋失败也可能直接从无锁/偏向跳到重量级锁，不总是逐级递进。
误区2：认为重量级锁一定性能差。在临界区执行时间很短且竞争不激烈时，自旋浪费CPU，重量级锁的阻塞唤醒可能更划算；现代JVM还有自适应自旋和锁粗化/消除，需结合场景分析。
思考题：为什么偏向锁撤销必须等待全局安全点（SafePoint）？请从线程栈中Lock Record的扫描、对象头原子修改以及程序执行正确性三个层面解释。
