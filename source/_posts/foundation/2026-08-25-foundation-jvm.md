---
title: "每日基础技术总结 · 2026-08-25 · JVM 对象头与锁升级"
date: 2026-08-25 08:00:00
categories: [技术分享]
tags: ["技术分享", "编程语言底层"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-25 · JVM 对象头与锁升级

## 📚 今日主题

> **JVM 对象头与锁升级**（编程语言底层）

### 1. 核心概念速览
JVM 对象头（Object Header）是堆中对象实例在内存布局中的前置元数据区域，用于存储对象运行时状态信息，包括标记字（Mark Word）和类型指针（Klass Pointer），在数组对象中还包括数组长度。锁升级（Lock Inflation）是指JVM基于内建监视器锁（Monitor）实现的synchronized在竞争加剧时，从无锁（无偏向）→偏向锁（Biased Locking）→轻量级锁（Thin Lock）→重量级锁（Fat Lock）的逐级膨胀过程。其本质是JVM在保证线程安全的前提下，根据竞争强度动态选择最经济的同步原语，以平衡CAS自旋、上下文切换与内核互斥的开销。该机制处于Java并发编程的底层支撑层，直接决定synchronized的吞吐量与延迟特性。专业工程师必须掌握它，因为所有高并发系统的锁优化、性能调优、死锁分析以及对‘synchronized是否仍有性能问题’的工程判断，均建立在对对象头布局与锁状态机转换的精确理解之上。

### 2. 底层原理剖析
对象头在64位JVM（默认启用压缩指针）中，普通对象由8字节Mark Word、4字节Klass Pointer组成（数组额外4字节长度）。Mark Word是锁状态的核心，其位分布随锁状态变化而复用：无锁态存储对象的identity hashcode、分代年龄、偏向标志位；偏向锁态存储持有偏向的线程ID、epoch、分代年龄；轻量级锁态存储指向栈中锁记录（Lock Record）的指针；重量级锁态存储指向ObjectMonitor的指针；GC标记态存储GC标记信息。锁升级路径的底层机制如下：

1. 无锁→偏向锁：当第一个线程访问同步块且JVM开启偏向锁（JDK15后默认禁用）时，JVM通过CAS将Mark Word中的偏向线程ID设为当前线程，并将偏向标志位置1。此后该线程进入同步块无需任何原子操作，仅检查线程ID是否匹配。

2. 偏向锁→轻量级锁：当另一个线程尝试获取偏向锁时，偏向模式撤销。撤销需等待全局安全点（SafePoint），偏向线程若已退出同步块则置为无锁；否则膨胀为轻量级锁。轻量级锁在当前线程栈帧中创建Lock Record，通过CAS将Mark Word替换为指向Lock Record的指针。若CAS成功则持有锁；失败说明有竞争，则自旋等待（适应性自旋）。

3. 轻量级锁→重量级锁：当自旋超过阈值或自旋期间有新线程竞争，JVM将Mark Word更新为指向ObjectMonitor的指针，进入重量级锁。ObjectMonitor内部维护EntryList、WaitSet以及owner线程，未获取锁的线程被挂起（park），依赖操作系统互斥量实现阻塞与唤醒。

与前端概念对比：Java的synchronized锁升级类似于浏览器事件循环中微任务与宏任务的调度层级，但更底层——它不是调度策略，而是内存布局与原子指令的联合决策。前端工程师熟悉的V8隐藏类（Hidden Class）与对象形状（Shape）在概念上类似JVM的对象头，但V8的隐藏类用于优化属性访问，不承担锁状态；而TS的interface仅是编译期类型约束，无运行时表示，与Java对象头中的Klass Pointer（运行时类型识别）有本质区别。锁升级的状态机与前端Redux的状态管理有同构性，但Redux状态迁移是纯函数，锁状态迁移依赖硬件原子指令与内存屏障。

### 3. 基础代码与实战验证
```text
以下为验证锁升级的极简Java代码，通过打印对象头二进制观察状态变化。需引入JOL（Java Object Layout）依赖，但核心逻辑不依赖框架。

public class LockUpgradeDemo {
    private static final Unsafe U = getUnsafe();

    public static void main(String[] args) throws Exception {
        Object obj = new Object();

        // 步骤1：无锁态，Mark Word通常为 0000000000000000000000000000000000000000000000000000000000000001
        System.out.println("无锁: " + markWord(obj));

        // 步骤2：计算identity hashcode（调用System.identityHashCode触发），Mark Word存储hashcode，无锁态变为001
        System.identityHashCode(obj);
        System.out.println("计算hash后: " + markWord(obj));

        // 步骤3：使用synchronized进入轻量级锁（若偏向锁未启用或已撤销）
        synchronized (obj) {
            // 轻量级锁的Mark Word指向当前线程栈帧中的Lock Record，最后两位为00
            System.out.println("轻量级锁: " + markWord(obj));
        }

        // 步骤4：模拟竞争触发膨胀，两个线程竞争同一把锁
        Thread t1 = new Thread(() -> {
            synchronized (obj) {
                // 持有锁并睡眠，让t2等待，此时锁可能已膨胀为重量级锁（最后两位为10）
                try { Thread.sleep(1000); } catch (InterruptedException e) {}
            }
        });
        Thread t2 = new Thread(() -> {
            synchronized (obj) {
                // 参与竞争
            }
        });
        t1.start();
        Thread.sleep(100); // 确保t1获取锁
        t2.start();
        t1.join(); t2.join();
        System.out.println("重量级锁: " + markWord(obj));
    }

    // 通过Unsafe获取对象的Mark Word（前8字节），转为二进制字符串
    private static String markWord(Object obj) {
        return Long.toBinaryString(U.getLong(obj, 0L));
    }

    private static Unsafe getUnsafe() {
        try {
            Field f = Unsafe.class.getDeclaredField("theUnsafe");
            f.setAccessible(true);
            return (Unsafe) f.get(null);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}

注释说明：Mark Word位于对象起始偏移0处，通过Unsafe.getLong直接读取原始8字节。锁状态由最后2位标记：01表示无锁或偏向锁（需进一步检查偏向位），00表示轻量级锁，10表示重量级锁，11表示GC标记。运行时应关闭偏向锁延迟（-XX:BiasedLockingStartupDelay=0）并开启偏向锁才能观察完整升级链。实际输出会因JVM版本与参数而异，但状态位规律不变。
```

### 4. 常见误区与进阶思考
误区1：认为锁升级是单向不可逆的。实际上偏向锁撤销后可能重新进入偏向模式（若JVM判定无竞争），轻量级锁膨胀为重量级锁后也可能在Monitor空闲后保持重量级，但对象头Mark Word在锁释放后可能恢复为无锁态，而非逆向降级为轻量级锁。锁的膨胀方向整体单向，但对象状态可回落到无锁，不能将‘升级’理解为状态只能递增。

误区2：认为偏向锁一定比轻量级锁快。偏向锁撤销需要全局安全点，若线程频繁交替执行同步块（而非同一线程重入），偏向锁的撤销开销远超轻量级锁的CAS自旋。因此JDK15后默认禁用偏向锁，并在JDK18中标记废弃。工程师不应盲目追求开启偏向锁，而应基于竞争频率与线程交替模式做实际基准测试。

思考题：在64位JVM上，若一个对象的Mark Word当前二进制最后三位为‘010’，请推断该对象处于什么锁状态？此时调用Object.wait()会发生什么？结合Mark Word的位布局解释为什么轻量级锁无法直接支持wait/notify，必须膨胀为重量级锁。
