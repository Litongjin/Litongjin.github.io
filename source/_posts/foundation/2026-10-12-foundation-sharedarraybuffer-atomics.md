---
title: "每日基础技术总结 · 2026-10-12 · Web Worker 中的 SharedArrayBuffer 与 Atomics 实现线程间同步的原理"
date: 2026-10-12 10:07:14
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-12 · Web Worker 中的 SharedArrayBuffer 与 Atomics 实现线程间同步的原理

## 📚 今日主题

> **Web Worker 中的 SharedArrayBuffer 与 Atomics 实现线程间同步的原理**（前端底层与计算机基础）

### 1. 核心概念速览
SharedArrayBuffer（SAB）是一块可以在多个 Web Worker 线程间直接共享的原始二进制内存缓冲区。它解决的问题是：通过 postMessage 传递 ArrayBuffer 时，要么结构化克隆产生副本，要么转移所有权导致原线程无法继续访问，两者都无法实现多线程对同一块内存的实时协作。SAB 配合 TypedArray 视图，使不同 Worker 能读写同一块内存，实现零拷贝共享。但共享内存引入数据竞争与内存可见性问题，Atomics 对象提供了对 Int32Array/BigInt64Array 的原子操作（add、sub、compareExchange、load、store）以及 wait/notify 阻塞/唤醒原语，本质上是将 CPU 原子指令与操作系统 futex 机制暴露给 JavaScript。该机制处于浏览器多线程编程的底层，与 C/C++ 的 std::atomic 和 POSIX 线程同步原语同源，是构建互斥锁、信号量、条件变量的基础。专业工程师必须掌握它，因为只有理解原子性、内存序与线程阻塞机制，才能正确实现高性能并发，并理解 Spectre 漏洞为何导致 SAB 需要跨源隔离。

### 2. 底层原理剖析
SharedArrayBuffer 在引擎内部映射到一块跨线程共享的内存区域（如 Linux 下通过 mmap 或 shm 实现）。每个 Worker 持有对该内存区域的引用，通过 Int32Array 等视图直接读写。由于多个线程可同时访问，ECMAScript 定义了同步内存模型：原子操作提供顺序一致性（seq-cst）保证，确保所有线程观察到的操作顺序一致。

Atomics 的原子操作由硬件保证：x86 使用 LOCK 前缀指令（如 LOCK XADD），ARM 使用 LDREX/STREX 独占加载/存储指令；这些指令在 CPU 缓存一致性协议（MESI）下确保可见性，并阻止指令重排。Atomics.add 在底层等价于一次不可分割的读-改-写。

Atomics.wait(ta, index, expected) 会将当前 Worker 线程挂起（注意：规范禁止在主线程使用，否则抛出 TypeError），底层调用操作系统 futex() 或 WaitOnAddress，将线程加入该内存地址的等待队列并让出 CPU。Atomics.notify(ta, index, count) 调用 futex wake，唤醒指定数量的等待线程。wait 会先原子地检查当前值是否等于 expected，若不相等则立即返回 'not-equal'，避免丢失唤醒信号；这个检查与挂起是一个原子操作（在内核中完成），从而防止竞态。

与前端已有概念的对比：postMessage 结构化克隆是数据复制或所有权转移，天然隔离但开销大，且无法同步感知修改；SharedArrayBuffer 是共享内存，零拷贝但要求程序员手动处理竞争。JS 单线程事件循环模型通过任务队列顺序执行，天然无数据竞争；引入 SAB 后，JS 获得了类似 C/C++ 多线程的共享内存语义。Atomics 类似于 C++ std::atomic（原子操作）与 std::condition_variable（wait/notify）的结合，而普通 JS 算术运算（如 ta[0] += 1）是非原子的，底层对应多条指令（load、add、store），在多线程下可能交错。

锁获取的伪代码逻辑：

lock(view):
  while (Atomics.compareExchange(view, 0, 0, 1) == 1):
    Atomics.wait(view, 0, 1)
unlock(view):
  Atomics.store(view, 0, 0)
  Atomics.notify(view, 0, 1)

compareExchange 原子地比较 view[0] 是否等于 0，若相等则写入 1 并返回旧值 0（获取锁成功）；若旧值为 1，表示锁被占用，调用 wait 挂起当前线程，等待锁值变为 0。被唤醒后重新尝试，处理伪唤醒与竞争。

### 3. 基础代码与实战验证
```text
// 主线程：创建共享内存，两个 Worker 并发递增计数器 1000 次
// 共享内存布局：index0 为锁变量（0=未锁定，1=锁定），index1 为计数器
const sab = new SharedArrayBuffer(Int32Array.BYTES_PER_ELEMENT * 2);
const ta = new Int32Array(sab);
ta[0] = 0; ta[1] = 0;

const workerSrc = `
  function lock(ta) {
    // compareExchange 原子比较并交换：如果 ta[0] 为 0，则置 1 并返回旧值 0（获取锁成功）；
    // 否则返回旧值 1（锁已被占用），进入 wait 阻塞，直到 unlock 通知后重试。
    while (Atomics.compareExchange(ta, 0, 0, 1) === 1) {
      Atomics.wait(ta, 0, 1); // 挂起当前线程，等待 ta[0] 不再为 1；被 notify 唤醒后返回
    }
  }
  function unlock(ta) {
    Atomics.store(ta, 0, 0); // 原子写回 0，并保证之前临界区写操作对其他线程可见（内存屏障）
    Atomics.notify(ta, 0, 1); // 唤醒一个等待锁的线程
  }
  self.onmessage = ({ data }) => {
    const ta = new Int32Array(data.sab);
    for (let i = 0; i < 1000; i++) {
      lock(ta);
      ta[1] += 1; // 非原子读-改-写，但受锁保护，不会发生竞争
      unlock(ta);
    }
    postMessage('done');
  };
`;

const blob = new Blob([workerSrc], { type: 'application/javascript' });
const url = URL.createObjectURL(blob);
let doneCount = 0;
const w1 = new Worker(url);
const w2 = new Worker(url);
w1.onmessage = w2.onmessage = () => {
  if (++doneCount === 2) {
    console.log('final counter:', ta[1]); // 应为 2000，证明锁同步有效
  }
};
w1.postMessage({ sab });
w2.postMessage({ sab });

// 注意：SharedArrayBuffer 需要在跨源隔离环境（COOP/COEP）下才能使用，否则构造器可能抛出异常。
```

### 4. 常见误区与进阶思考
常见误区：
1. 误以为对 SharedArrayBuffer 上的 TypedArray 元素进行普通读写或自增（如 ta[0]++）是线程安全的。实际上这些操作编译为多条机器指令，多线程并发执行会互相交错，产生数据竞争，且不保证内存可见性。必须使用 Atomics 提供的方法，或将共享内存的读写放入由锁保护的临界区。
2. 误以为 Atomics.notify 会像消息一样排队：如果在调用 notify 时没有线程正在 Atomics.wait 该地址，唤醒信号会直接丢失，后续再调用 wait 的线程会一直等待。因此典型模式是：等待方先检查状态再决定是否 wait，并且 wait 的 expected 参数必须与当前状态匹配；通知方修改状态后调用 notify；等待方被唤醒后需要重新检查状态。

进阶思考：在锁实现中，为什么被 Atomics.notify 唤醒的线程不能直接认为锁已可用，而必须回到 while 循环重新调用 Atomics.compareExchange 检查锁状态？请从伪唤醒、多线程竞争和锁的非所有权转让角度分析。
