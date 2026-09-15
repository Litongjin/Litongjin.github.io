---
title: "每日基础技术总结 · 2026-09-15 · JVM 对象头与锁升级"
date: 2026-09-15 08:00:00
categories: [技术分享]
tags: ["技术分享", "编程语言底层"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-15 · JVM 对象头与锁升级

## 📚 今日主题

> **JVM 对象头与锁升级**（编程语言底层）

### 1. 核心概念速览
### 本质定义

**对象头（Object Header）**：HotSpot 为每个堆上对象分配的固定前缀字节区，64 位 JVM 下布局为 `Mark Word(8B) | Klass Pointer(4B 压缩 / 8B 不压缩) | [array length(4B，仅数组)]`。其中 **Mark Word** 是一个**按同步状态与 GC 状态复用位域的机器字**，其低 2 位 `lock` 是状态标签：`01` 无锁或偏向、`00` 轻量级锁、`10` 重量级锁、`11` GC 标记（转发指针/markOop）。

**锁升级**：`monitorenter/monitorexit` 的运行时实现并不固定指向操作系统互斥量。HotSpot 依据**运行时观测到的竞争强度**在三段路径间迁移：偏向（重入判定退化为一次 `load + cmp`，无原子指令）→ 轻量级（CAS 把线程栈帧内 Lock Record 的地址写入 Mark Word）→ 重量级（膨胀为 C++ 侧的 `ObjectMonitor`，竞争失败者 `park` 陷入内核态）。所谓“升级”，其物理动作就是**改写 Mark Word 的位域或指针字段**。

### 它解决什么问题

Java 语言规范要求每个对象内建监视器语义（`synchronized`、`Object.wait/notify`）。但真实负载中绝大多数锁在其生命周期内**从无真实竞争**。若 `monitorenter` 一律走内核互斥量，则每次同步都需支付用户态/内核态切换、调度器介入与上下文切换成本（微秒量级），而原子 CAS 是纳秒量级——两个数量级的差距。锁升级把「同步状态的存储」下沉到对象自身的内存字里，使无竞争路径**全程停留在用户态**，仅当竞争真实发生时才付出内核代价。这是典型的**基于运行时反馈的自适应优化**：把状态机的状态位放进被保护对象本身，避免为每个对象外挂同步元数据。

### 在计算机体系中的位置

- **向下**：与 CPU 原子指令（x86 `LOCK CMPXCHG`、ARM `LDXR/STXR`）、内存屏障、Linux `futex` 直接对接（`Unsafe.park` → `pthread_mutex_lock` → `futex`）。
- **平级**：与 GC 强耦合。Mark Word 在 GC 停顿期被复用为转发指针/markOop，`age:4` 即分代年龄。锁状态与 GC 状态共享同一个字，因此膨胀（inflate）与撤销（revoke）必须在**安全点（safepoint）**上与 GC 协调。
- **向上**：`synchronized` 语义与 happens-before、`Object.identityHashCode`、`jstack` 的锁信息、JFR 的 `JavaMonitorEnter`/`JavaMonitorInflate` 事件；`ReentrantLock` 也通过 AQS → `LockSupport.park/unpark` 落到同一套 park/unpark 原语。
- **内存开销**：对象头决定单对象最小尺寸（压缩指针下 `new Object()` = 16B：12B 头 + 4B padding），直接影响堆占用、GC 频率与 cache 局部性——对推理服务、特征工程这类对象密集型负载是容量与尾延迟的一阶因素。

### 为什么专业工程师必须掌握

线程池/Reactor 的尾延迟抖动、`-XX:-UseBiasedLocking` 的工程取舍、JFR 里 `JavaMonitorEnter` 与 inflate 事件的解读、以及“为何某些场景 `ReentrantLock` 优于 `synchronized`”，全部只有在 Mark Word 的状态机与 `ObjectMonitor` 的队列结构上才能解释清楚。它是 Java 内存模型从语言规范落到硬件指令的最后一跳。

### 2. 底层原理剖析
### 一、对象内存布局与对齐

`| Mark Word (8B) | Klass Pointer (4B/8B) | [array length (4B)] | instance fields | padding |`

- `Klass Pointer` 指向方法区中的 `InstanceKlass`，是**运行时类型身份的唯一依据**：`getClass()`、`instanceof`、`invokevirtual` 的 vtable/itable 分派、JIT 的类型剖面都从它出发。默认 `-XX:+UseCompressedOops` 时占 4B（堆 < 32G）。
- 字段重排序：HotSpot 按 `long/double → int/float → short/char → byte/boolean → oops` 排列以最小化 padding；对象起始地址满足 8 字节对齐，故尾部补齐。

### 二、Mark Word 位域（64 位 HotSpot，低位在右）

    无锁     : unused:25 | identity_hashcode:31 | unused:1 | age:4 | biased_lock:0 | lock:01
    偏向     : JavaThread*:54 | epoch:2 | unused:1 | age:4 | biased_lock:1 | lock:01
    轻量级   : ptr_to_LockRecord:62 | lock:00
    重量级   : ptr_to_ObjectMonitor:62 | lock:10
    GC 标记  : forwarding/markOop 信息 | lock:11

从低位到高位依次是：`lock(bit0-1)`、`biased_lock(bit2)`、`age(bit3-6)`、`unused(bit7)`、`identity_hashcode(bit8-38)`。**位域复用是核心设计**：同一个 8 字节承载可选的状态，使每对象固定开销最小化。由此推出两个硬约束：

1. `identity_hashcode` 与偏向锁的 `JavaThread*` 位域互斥——对象一旦计算过 identity hash，就永远无法进入偏向状态；
2. 偏向对象若被调用 `hashCode()`，必须先撤销偏向（revoke）才能写入 hash。

### 三、轻量级锁：CAS + 线程栈上的 Lock Record

    monitorenter(obj):
        lr = 当前栈帧分配 BasicObjectLock        // 不占堆，随栈帧回收
        lr._displaced_header = obj.mark          // 保存原 Mark Word
        if CAS(&obj.mark, obj.mark, lr | 0b00):  // 抢占成功，Mark Word 指向 lr
            return
        if (obj.mark & ~0b11) == lr:             // 指向本线程的 Lock Record → 重入
            push(LockRecord(_displaced_header = 0))   // 0 作为“重入”哨兵值
            return
        if 其他线程持有或 CAS 持续失败:
            inflate(obj)                          // 膨胀为 ObjectMonitor
            ObjectMonitor::enter(obj.monitor)

    monitorexit(obj):
        lr = pop()
        if lr._displaced_header == 0: return      // 重入退出一层，Mark Word 不动
        CAS(&obj.mark, lr | 0b00, lr._displaced_header)   // 最外层退出，写回原值
        if CAS 失败: ObjectMonitor::exit(...)     // 已被膨胀，走重量级路径

关键点：**重入时对象头完全不变**，重入深度由线程栈上 Lock Record 的数量隐式记录（`_displaced_header == 0` 的个数）。轻量级锁的“自旋”本质是 CAS 重试，是**乐观并发**：假设临界区极短、竞争窗口极小。

### 四、重量级锁：ObjectMonitor 与 park

    ObjectMonitor: { _owner, _recursions, _cxq, _EntryList, _WaitSet, _count, _succ }

    enter(m):
        if CAS(_owner, NULL, self): return          // 无竞争快路径：仍是用户态 CAS（重量级锁也有快路径）
        if _owner == self: _recursions++; return    // 重入，O(1)
        // 真竞争：入队 + 阻塞
        CAS 把当前线程节点压入 _cxq
        for (;;):
            if 尝试获取成功: return
            if _succ == self: _succ = NULL
            park()                                   // futex_wait → 线程置为 TASK_INTERRUPTIBLE，让出 CPU

    exit(m):
        if _owner != self: throw IllegalMonitorStateException
        if --_recursions > 0: return
        _owner = NULL; 必要时 unpark(_EntryList/_cxq 的队首)   // futex_wake

代价来源明确：`park` 是系统调用 + 调度器介入，唤醒后还要重新竞争，延迟从纳秒跳到微秒并伴随抖动。因此“重量级”不是实现差，而是竞争激烈时**唯一不浪费 CPU 的选择**（自旋在竞争下只会空转烧核）。

### 五、偏向锁与批量重偏向/批量撤销

- 首次加锁：CAS 把 `JavaThread*` 与 `epoch` 写入 Mark Word；此后同一线程重入仅做 `mark.thread == self` 比较。
- 撤销（revoke）：另一线程尝试加锁或本线程调用 `hashCode()` 时触发。撤销**必须等到偏向线程到达安全点**，由 VM 遍历其栈帧找到对应 Lock Record 并改写 Mark Word——这意味着一次 STW 级别的停顿。
- `epoch` 机制：当某个 `InstanceKlass` 的撤销次数超过 `BiasedLockingBulkRebiasThreshold`（默认 20），JVM 递增该类原型头的 epoch，使已过期的偏向对象在下次加锁时**可直接重偏向**而不必逐个撤销；撤销超过 `BiasedLockingBulkRevokeThreshold`（默认 40），该类被永久标记为不可偏向（bulk revoke）。
- `BiasedLockingStartupDelay` 默认 4000ms：JVM 启动后 4 秒才启用偏向锁，规避启动期（大量 JVM 内部类初始化）的撤销风暴。
- **JDK 15（JEP 374）起默认关闭偏向锁，后续版本已移除该实现**，现代 HotSpot 实际只剩“轻量级 ↔ 重量级”两级。

### 六、与前端已有知识体系的对照

| 概念 | Java / HotSpot | JavaScript / V8 |
|---|---|---|
| 运行时类型指针 | `Klass Pointer` → InstanceKlass/vtable | 对象首部的 hidden class(map) 指针 |
| 方法分派优化 | 类型剖面 → 单态内联 → 去优化 | Inline Cache：monomorphic → polymorphic → megamorphic 退化 |
| 接口 | 运行时实体，`invokeinterface` 查 itable | TS 的 `interface` 是**编译期结构类型，运行时被完全擦除**，不存在任何元数据，`x instanceof I` 不合法 |
| 共享内存同步 | Mark Word + ObjectMonitor + futex | 单线程事件循环天然串行，无锁；仅 `SharedArrayBuffer + Atomics` 引入共享可变内存，`Atomics.wait` 语义上等价于 `park`，实现层同样落到 futex |
| 位运算心智 | Mark Word 是 64 位掩码字 | JS 位运算被 ToInt32 截断为 32 位有符号数，**无法用位运算解析完整 64 位 Mark Word**，需 `BigInt` 或直接读 long |

最本质的差异：前端没有“对象内建同步状态”这一层，因为 JS 的可变状态共享被事件循环消除了；一旦引入 SAB，就必须重新发明一套与 Mark Word 同构的机制。理解对象头，等价于理解“在共享内存模型下，状态位必须存在被保护对象自身的元数据中”这一设计约束。

### 3. 基础代码与实战验证
```text
### 验证一：直接读取并解码 Mark Word（JDK 8~17，单文件，无第三方依赖）

运行：`java MarkWordDemo`（观察轻量级锁请加 `-XX:-UseBiasedLocking`）

    import sun.misc.Unsafe;
    import java.lang.reflect.Field;

    public class MarkWordDemo {
        static final Unsafe U;
        static {
            try {
                Field f = Unsafe.class.getDeclaredField("theUnsafe");
                f.setAccessible(true);
                U = (Unsafe) f.get(null);
            } catch (Exception e) { throw new ExceptionInInitializerError(e); }
        }

        static long   mark(Object o)  { return U.getLong(o, 0L); }   // 偏移 0 即 Mark Word，与 Klass Pointer 的字段顺序由 VM 固定
        static String hex(long v)     { return String.format("0x%016X", v); }
        static int    lockBits(long v){ return (int) (v & 0b11); }    // 低 2 位 = 锁状态标签
        static int    age(long v)     { return (int) ((v >> 3) & 0b1111); } // bit3-6 = 分代年龄

        public static void main(String[] args) throws Exception {
            Object o = new Object();

            // 1) 新建对象：lock=01。biased_lock 位取决于是否已过 BiasedLockingStartupDelay(默认4000ms)
            long m0 = mark(o);
            System.out.println("new        " + hex(m0) + " lock=" + lockBits(m0) + " age=" + age(m0));

            // 2) 计算 identity hash：31 位哈希被写进 Mark Word 的 bit8-38，
            //    该位域与偏向锁的 JavaThread* 互斥 —— 此对象从此不可能再进入偏向状态
            int h = System.identityHashCode(o);
            long m1 = mark(o);
            System.out.println("hashCode   " + hex(m1) + " hash=" + Integer.toHexString(h) + " lock=" + lockBits(m1));

            // 3) 无竞争进入 synchronized：CAS 把当前线程栈帧内 Lock Record 的地址写入 Mark Word，
            //    原 Mark Word(m1) 被保存在该 Lock Record 的 _displaced_header 中（不在堆上）
            synchronized (o) {
                long m2 = mark(o);
                System.out.println("locked     " + hex(m2) + " lock=" + lockBits(m2)); // 期望 lock=00

                // 4) 重入：Mark Word 不再变化，只在栈上再压入一个 Lock Record，
                //    其 _displaced_header = 0，用哨兵值 0 把“层数”编码进栈而非对象头
                synchronized (o) {
                    System.out.println("reentrant  " + hex(mark(o)) + " lock=" + lockBits(mark(o)));
                }
            }

            // 5) 最外层 monitorexit：CAS 把 _displaced_header 写回，对象回到无锁
            long m3 = mark(o);
            System.out.println("unlocked   " + hex(m3) + " lock=" + lockBits(m3));
        }
    }

预期输出形态（hash 段随机）：`new 0x0000000000000001 lock=1` → `hashCode 0x0000000XXXXXX801`（hash 落在 bit8 以上）→ `locked 0x00007F...0 lock=0`（指向栈地址）→ `reentrant` 与 `locked` 完全相同 → `unlocked` 回到 bit8 以上的 hash 值、`lock=01`。

### 验证二：竞争导致膨胀为 ObjectMonitor（Mark Word 低 2 位变 10，值变成 C++ 对象指针）

    import sun.misc.Unsafe;
    import java.lang.reflect.Field;

    public class InflateDemo {
        static final Unsafe U;
        static {
            try { Field f = Unsafe.class.getDeclaredField("theUnsafe"); f.setAccessible(true);
                  U = (Unsafe) f.get(null); } catch (Exception e) { throw new Error(e); }
        }
        public static void main(String[] args) throws Exception {
            final Object lock = new Object();
            synchronized (lock) {                                  // 主线程先持有（轻量级或偏向）
                Thread t = new Thread(() -> {
                    synchronized (lock) { }                        // t 在此阻塞：CAS 失败且非重入 → inflate
                });
                t.start();
                Thread.sleep(500);                                 // 确保 t 已 park 在 ObjectMonitor 的 _cxq 上
                // lock 变量逃逸到线程 t，JIT 无法做锁消除，观察到的必然是真实状态
                long m = U.getLong(lock, 0L);
                System.out.printf("mark=0x%016X lock=%d%n", m, (int)(m & 0b11)); // lock=10
            }
            // 释放后线程 t 被 unpark 唤醒，重新竞争，Mark Word 才可能被 deflate
        }
    }

### 验证三：偏向量化统计（JDK 8~14）

    java -XX:+UnlockDiagnosticVMOptions -XX:+PrintBiasedLockingStatistics \
         -XX:BiasedLockingBulkRebiasThreshold=20 \
         -XX:BiasedLockingBulkRevokeThreshold=40 MarkWordDemo

输出中 `revoked #` / `bulk revoke` 计数可直接观测“撤销即 STW”的代价分布。
```

### 4. 常见误区与进阶思考
### 误区一：把锁升级当作“不可逆的单向阶梯”，并默认偏向锁始终可用

两个独立错误叠加：

1. **可逆性**：重量级锁并非终点。HotSpot 会在安全点上尝试 monitor deflation（`ObjectSynchronizer::deflate_idle_monitors`），把空闲的 ObjectMonitor 归还并按情况回退；偏向锁也存在批量重偏向（epoch 递增）让已过期偏向重新生效。把它理解为**状态机上的带条件迁移**，而非阶梯，才是正确的模型。
2. **默认可用性**：`BiasedLockingStartupDelay` 默认 4000ms，启动后 4 秒才生效；JDK 15（JEP 374）起默认 `-XX:-UseBiasedLocking`，后续版本已移除实现。在线程池 + 高并发场景下，同类对象被多个线程交替加锁会触发撤销，而撤销必须等到偏向线程到达安全点，造成的 STW 开销往往远大于它省下的那次 CAS——这正是它被废弃的工程原因。

### 误区二：认为“轻量级锁一定优于重量级锁”，且忽略 Mark Word 位域互斥与 JIT 侧优化

- **“轻量级优于重量级”是伪命题**。轻量级的自旋是有界的乐观重试：竞争激烈或临界区较长时，自旋只消耗 CPU 且延迟无界，此时膨胀并在队列上 park 才是正确选择。脱离竞争强度谈优劣没有意义。
- **位域互斥的连锁反应**：对象只要调用过 `identityHashCode`/默认 `hashCode()`，偏向锁位域已被 hash 占用，该对象永不可偏向；反之偏向中的对象一旦调用 `hashCode()` 会触发撤销。把 hash 当作“纯读操作”是典型误判。
- **混淆 JIT 优化与运行时升级**：`synchronized` 在 JIT 侧还会经历**锁消除**（逃逸分析证明锁对象不逃逸，结合标量替换直接删掉 `monitorenter/exit`）与**锁粗化**（把相邻的多次加解锁合并）。这些发生在编译期，与 Mark Word 的状态机无关。`jstack` 看不到锁、性能却很好，往往就是锁消除的结果。
- **内存常识缺失**：`new Object()` 在开启压缩指针时占 16 字节（12 字节头 + 4 字节 padding），字符串/包装类都带头；对象头是导致“小对象海量分配”内存放大与 GC 压力的隐性成本源。

### 思考题

场景：8 个线程共享一个 `ConcurrentHashMap<Class<?>, Object>`，每个线程通过 `computeIfAbsent` 拿到**自己专属**的锁对象，然后 `synchronized` 该对象。任意时刻不存在两个线程争抢同一个锁，但 8 个线程会持续、高频地**交替**对同一批对象加锁（线程 A 在 t1 拿 lockX，线程 B 在 t2 拿 lockX……）。

请回答：

1. 在 `-XX:+UseBiasedLocking` 且已过 4 秒延迟的前提下，推导该批对象 Mark Word 的演进轨迹：偏向 A → 偏向撤销 → 偏向 B → …… 并说明 `epoch` 位在什么时刻被递增、`lock`/`biased_lock` 位如何变化；
2. 为什么这种**“无真实竞争”**的负载最终会触发 `bulk revoke`（默认阈值 40），使整个类的所有对象被永久标记为不可偏向？请从“撤销必须等到偏向线程到达安全点”这一约束出发，说明为什么撤销不能在持有者运行时异步完成（提示：需要读取并修改持有者**栈上**的 `BasicObjectLock._displaced_header`，而栈帧布局在非安全点处不可信）；
3. 给定以上分析，说明为什么在**线程池/异步框架**这类“锁对象的归属频繁在线程间迁移”的负载上，`-XX:-UseBiasedLocking`（或直接升级到 JDK 15+）反而是更优配置，而在“单线程长生命周期持有同一个锁”的负载上偏向锁仍有收益。

如果一个工程师能准确画出 3 个时刻的 Mark Word 十六进制形状并解释每一次 safepoint 的触发原因，说明他已经把对象头、safepoint 与 GC 三者的耦合关系打通了。
