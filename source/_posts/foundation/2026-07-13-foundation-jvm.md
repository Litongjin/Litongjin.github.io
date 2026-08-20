---
title: "每日基础技术总结 · 2026-07-13 · JVM 对象头与锁升级"
date: 2026-07-13 08:00:00
categories: [技术分享]
tags: ["技术分享", "编程语言底层"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-13 · JVM 对象头与锁升级

## 📚 今日主题

> **JVM 对象头与锁升级**（编程语言底层）

### 1. 核心概念速览
JVM 对象头（Object Header）是堆中每个对象实例在内存布局最前端的固定元数据区，HotSpot VM 中通常由 Mark Word（标记字）和 Klass Pointer（类型指针，在压缩指针开启时可能被压缩或省略）组成，数组对象额外包含数组长度。Mark Word 在 64 位 JVM 中占 8 字节，其二进制位模式被设计为一种可复用的自描述状态机，用于存储对象的哈希码、GC 分代年龄、锁状态标志、指向锁记录的指针、指向重量级监视器（Monitor）的指针等。锁升级（Lock Inflation）指的是 synchronized 在无竞争、轻度竞争、重度竞争下，锁状态沿 无锁（Normal）→ 偏向锁（Biased）→ 轻量级锁（Lightweight Locked）→ 重量级锁（Heavyweight Locked）单向演化的过程，且状态只能升级、不能降级（除偏向锁可被撤销回无锁/轻量级）。该机制解决的核心问题是：多线程互斥同步在无竞争或低竞争场景下，避免直接调用操作系统互斥量（pthread_mutex/futex）带来的用户态/内核态切换开销，同时保持语义正确性。它在 JVM 整体架构中属于运行时数据区与执行引擎的交叉领域，是理解 Java 并发编程底层语义、锁性能调优、以及 JFR/JMX 锁指标含义的基石。专业工程师必须掌握它，因为锁是并发正确性的基础原语，而对象头是锁状态的物理载体；脱离对象头谈 synchronized 的‘可重入’‘可见性’‘内存屏障’都只是空中楼阁。前端工程师熟悉的 JavaScript 事件循环是单线程协作式调度，无锁冲突概念；而 Java 的 synchronized 是基于共享内存多线程模型，锁升级本质上是对‘同步原语成本’的分级递进策略——类似浏览器渲染引擎对 DOM 操作批量合并、或 V8 对对象形状（Shape）内联缓存的优化思路：用精细的运行时状态位换来常见场景的高效执行。

### 2. 底层原理剖析
对象头的精确布局与状态机：
1. Mark Word（64 位）：低 2 位（或 3 位，取决于是否开启偏向锁）作为锁状态标志位。不同锁状态下的位含义：
   - 无锁（01）：高位存 31 位无偏哈希码（identity hash code）、1 位 unused、4 位分代年龄（age）、1 位 biased_lock=0、1 位偏向标记=0、2 位锁标志=01。注意哈希码只有在调用 System.identityHashCode() 或 Object.hashCode()（未重写时）才写入，且一旦写入则对象无法进入偏向锁（因为偏向锁需要复用这些位存线程 ID）。
   - 偏向锁（101）：biased_lock=1、锁标志=01；位域存持有偏向线程的 JavaThread*（54 位）、epoch（偏向纪元，2 位）、分代年龄（4 位）。线程 ID 是 JVM 内部线程结构指针的低位。偏向锁的获得只需 CAS 将空线程 ID 写为自己；可重入只需检查当前线程 ID 相同即可，无任何原子操作。
   - 轻量级锁（00）：Mark Word 整体存指向栈中 Lock Record 的指针（64 位）。线程在栈帧中分配 Lock Record，通过 CAS 将对象头中的 Mark Word 复制到 Lock Record 中，并尝试将对象头替换为指向 Lock Record 的指针。CAS 失败或已有竞争者，则膨胀。
   - 重量级锁（10）：Mark Word 存指向 ObjectMonitor 的指针（64 位）。ObjectMonitor 内部包含 _owner 线程、_recursions（重入计数）、_WaitSet、_EntryList、cxq 等，调用操作系统 futex 阻塞/唤醒。
   - GC 标记（11）：用于标记对象可回收，与锁无关。
2. Klass Pointer（类型指针）：指向方法区中的 Klass 对象，用于确定对象的类型。开启压缩指针（-XX:+UseCompressedOops）时占 4 字节；否则 8 字节。如果 JVM 使用压缩类指针（-XX:+UseCompressedClassPointers），且堆内对象的 Klass 指针可以被压缩到 4 字节，则对象头可能为 12 字节（8 字节 Mark Word + 4 字节 Klass Pointer），但需要按 8 字节对齐填充到 16 字节。数组对象额外加 4 字节长度字段，然后填充对齐。
3. 锁升级的触发条件（HotSpot 实现细节）：
   - 偏向锁（默认开启，-XX:+UseBiasedLocking）：首次有线程进入 synchronized 块时，如果对象头是 可偏向 且 thread ID 为空，则 CAS 写入当前线程 ID，偏向锁生效。之后该线程再次进入无需任何同步操作。只有当另一个线程尝试获取偏向锁时，才发生偏向撤销（revoke）。
   - 偏向撤销：当竞争发生时，先到达安全点（SafePoint），暂停线程，判断偏向线程是否存活且仍处于临界区；如果已退出临界区，则重置对象头为无锁；如果仍活跃，则升级为轻量级锁（或直接重量级锁，取决于竞争烈度）。撤销后如果对象很快再次被竞争，可能直接进入轻量级锁。
   - 轻量级锁：线程 A 获得偏向锁后，B 尝试获取。若 A 已释放，则对象可能回到无锁或不可偏向状态；B 通过 CAS 设置指向自身 Lock Record 的指针。如果 CAS 成功，B 持有轻量级锁；如果失败，说明 A 与 B 同时竞争（或 A 未释放），则轻量级锁膨胀。
   - 膨胀（Inflate）：当多个线程同时竞争轻量级锁时，JVM 创建 ObjectMonitor，并 CAS 将对象头替换为指向 Monitor 的指针。后续所有线程（包括原持有者）都通过 Monitor 的 enter/exit 方法竞争，未获得锁的线程被挂起进入内核等待队列。重量级锁的阻塞/唤醒涉及系统调用，成本最高。
   - 锁状态只能升级不能降级：JVM 不会主动将重量级锁降回轻量级或偏向锁。这是为了减少复杂度和避免抖动；但偏向锁在安全点可以批量重偏向（bulk rebias）或批量撤销（bulk revoke），这是通过 epoch 位实现的。
4. 与前端概念的异同：
   - 相似点：JavaScript 对象的隐藏类（Hidden Class / Map）与 JVM 对象头一样，都承载元数据；V8 的指针压缩（Pointer Compression）与 JVM 的 CompressedOops 类似，都是为了减少内存占用。前端中 React 的虚拟 DOM 类型标记、Vue 的响应式依赖追踪，本质上都是通过位标记/状态字段来优化运行时行为。
   - 差异：JS 没有多线程共享内存，所有‘锁’概念被事件循环取代；TS 的 interface 是编译期类型结构，与运行时无关，而 Java 的 synchronized 锁对象是运行时对象头的状态；TS 的 interface 可以理解为类型层面的约束，而 Java 的对象头是实例层面的物理状态。前端开发中的‘锁’一般指互斥量（如 Web Locks API），但那是浏览器提供的独立机制，与 JS 引擎内存模型无关。
5. 伪代码描述（锁获取核心逻辑）：
   if (biased_lock_enabled && obj->mark_word.biased_lock == 1 && obj->mark_word.thread_id == NULL) {
       CAS(obj->mark_word, current_thread_id);
       // 成功则进入偏向模式；失败则竞争升级
   } else if (obj->mark_word.lock_bits == 01 && obj->mark_word.biased_lock == 1) {
       if (obj->mark_word.thread_id == current_thread_id) {
           // 重入，直接进入临界区，无任何同步开销
       } else {
           // 触发偏向撤销，可能升级
       }
   } else if (obj->mark_word.lock_bits == 00) {
       // 轻量级锁：复制 Mark Word 到栈中的 Lock Record，CAS 指向自己
       LockRecord* lr = allocate_lock_record();
       lr->displaced_mark = obj->mark_word;
       CAS(obj->mark_word, lr);
       if (CAS 成功) 持有轻量级锁;
       else inflate();
   } else if (obj->mark_word.lock_bits == 10) {
       // 重量级锁：进入 ObjectMonitor，可能阻塞
       ObjectMonitor* mon = obj->mark_word.monitor;
       mon->enter(current_thread);
   }

注意：偏向锁在 JDK 15 之后默认被禁用（JEP 374），并在 JDK 18 中标记为废弃（JEP 394 提议），原因是现代应用竞争普遍，偏向锁维护成本高。但锁升级的轻量级→重量级路径仍然存在且重要。

### 3. 基础代码与实战验证
```text
以下为纯 Java 代码，不依赖框架，用于验证锁升级过程中对象头状态的变化。注意：偏向锁在 JDK 15+ 默认关闭，需显式启用 -XX:+UseBiasedLocking；且需要 JOL（Java Object Layout）工具类查看对象头。这里给出核心验证思路和代码。

import org.openjdk.jol.info.ClassLayout;

public class LockUpgradeDemo {
    static final Object lock = new Object();

    public static void main(String[] args) throws Exception {
        // 1. 无锁状态：打印对象头，注意 Mark Word 中的锁标志位为 01
        System.out.println("=== 无锁 ===");
        System.out.println(ClassLayout.parseInstance(lock).toPrintable());

        // 2. 触发 hashcode 计算，观察偏向锁被禁用（hashcode 写入对象头，无法偏向）
        System.out.println("hashcode: " + System.identityHashCode(lock));
        System.out.println("=== 计算 hashcode 后 ===");
        System.out.println(ClassLayout.parseInstance(lock).toPrintable());

        // 3. 偏向锁（需 JDK < 15 且开启偏向锁；如果不可用则跳过此步骤）
        synchronized (lock) {
            System.out.println("=== 偏向锁（如果支持） ===");
            System.out.println(ClassLayout.parseInstance(lock).toPrintable());
        }

        // 4. 轻量级锁：启动一个线程，用两个线程交替竞争（低竞争）
        Thread t1 = new Thread(() -> {
            synchronized (lock) {
                System.out.println("=== 轻量级锁（t1 持有） ===");
                System.out.println(ClassLayout.parseInstance(lock).toPrintable());
                try { Thread.sleep(100); } catch (InterruptedException e) {}
            }
        });
        t1.start();
        t1.join();

        // 5. 重量级锁：两个线程同时激烈竞争，让线程长时间持有锁，制造竞争
        Thread t2 = new Thread(() -> {
            synchronized (lock) {
                System.out.println("=== 重量级锁（t2 持有） ===");
                System.out.println(ClassLayout.parseInstance(lock).toPrintable());
                try { Thread.sleep(500); } catch (InterruptedException e) {}
            }
        });
        Thread t3 = new Thread(() -> {
            synchronized (lock) {
                System.out.println("=== 竞争线程 t3 ===");
                System.out.println(ClassLayout.parseInstance(lock).toPrintable());
            }
        });
        t2.start();
        Thread.sleep(50); // 确保 t2 先拿到锁
        t3.start();
        t2.join();
        t3.join();

        // 6. 最终状态：锁不会降级，仍是重量级锁
        System.out.println("=== 最终状态 ===");
        System.out.println(ClassLayout.parseInstance(lock).toPrintable());
    }
}

// 预期关键输出（注释说明底层）：
// 无锁：mark word 最低位为 01，且不包含线程 ID。
// 计算 hashcode 后：mark word 中 hashcode 位被填满，biased_lock 位变为 0，表示对象不可偏向。
// 偏向锁：mark word 最低三位为 101，且 thread ID 为当前线程。
// 轻量级锁：mark word 最低两位为 00，高位为指向线程栈中 Lock Record 的指针。
// 重量级锁：mark word 最低两位为 10，高位为指向 ObjectMonitor 的指针。
// 验证升级不可逆：最终状态仍为 10。

若无法使用 JOL，可用以下文字化步骤描述锁升级过程（等价于伪代码）：
1. 创建普通对象，Mark Word 为 0x01（无锁，可偏向）。
2. 线程 A 进入 synchronized 块，JVM 检查 biased_lock=1，thread_id=0，CAS 将线程 A ID 写入 Mark Word，状态变为 101。
3. 线程 A 重入时，检查 thread_id 匹配，不做任何操作。
4. 线程 B 尝试进入，发现 thread_id 不同，触发偏向撤销。若 A 已退出临界区，则 Mark Word 恢复为 01；若 A 未退出，则升级为轻量级锁（00），B 和 A 竞争 CAS 设置 Lock Record 指针。
5. 若竞争加剧（多个线程自旋失败或自旋超时），JVM 创建 ObjectMonitor，将 Mark Word 改为指向 Monitor 的指针（10），后续线程通过 Monitor 的 enter 方法竞争，未获得锁的线程进入内核等待队列。
6. 锁一旦升级到 10，即使竞争消失，也不会降级回 00 或 01。
```

### 4. 常见误区与进阶思考
误区 1：认为 synchronized 只是简单加锁/解锁，不理解锁状态是对象头中的位模式，导致无法解释为什么调用 hashCode() 会影响锁性能、为什么偏向锁在某些场景下反而降低吞吐、为什么锁无法降级。实际：JVM 的锁机制是基于对象头位域的状态机，任何修改对象头的操作（如计算 identity hashcode）都会影响后续锁状态路径；偏向锁的撤销需要安全点，大量偏向锁撤销会导致 Stop-The-World 暂停，所以 JDK 15+ 默认禁用偏向锁。
误区 2：认为锁升级是 JVM 自动优化的免费午餐，忽略了锁消除（Lock Elision）和锁粗化（Lock Coarsening）的区别，以及自旋（Spin）与重量级锁的关系。很多人以为轻量级锁失败就立即膨胀，实际上 HotSpot 在轻量级锁失败后会先进行自适应自旋（Adaptive Spinning），自旋失败才膨胀为重量级锁。自旋在轻量级锁和重量级锁获取路径中都有应用，但 JVM 内部有独立的 spin 计数器和策略，且现代 JVM 在偏向锁撤销时也可能直接进入重量级锁而跳过轻量级锁（如果竞争者多于一个）。
深度思考题：如果在一个对象上先调用 System.identityHashCode(obj)，然后该对象被两个线程交替但非竞争地访问（即同一时刻只有一个线程进入 synchronized 块），请问该对象的锁状态会经历哪些步骤？为什么 hashcode 的存在会导致偏向锁失效？进一步，假设你禁用偏向锁（-XX:-UseBiasedLocking），同样的访问模式下，锁状态会在轻量级锁和无锁之间如何切换？请从 Mark Word 位域复用的角度给出精确解释，并思考为什么 JVM 在对象被计算过 hashcode 后不能再进入偏向锁状态。
