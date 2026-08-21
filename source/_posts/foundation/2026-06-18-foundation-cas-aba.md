---
title: "每日基础技术总结 · 2026-06-18 · 无锁编程：CAS 循环与 ABA 问题解决"
date: 2026-06-18 08:00:00
categories: [技术分享]
tags: ["技术分享", "操作系统基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-06-18 · 无锁编程：CAS 循环与 ABA 问题解决

## 📚 今日主题

> **无锁编程：CAS 循环与 ABA 问题解决**（操作系统基础）

### 1. 核心概念速览
无锁编程（Lock-Free Programming）指不依赖操作系统互斥锁，仅通过原子硬件指令（如 CAS、LL/SC）协调多线程并发访问共享内存。CAS（Compare-And-Swap）是核心原语：它原子地比较内存地址中的当前值与期望值，若相等则替换为新值并返回真，否则不修改并返回假。基于 CAS 的循环（retry loop）通过反复读取当前值、计算新值、尝试 CAS 直至成功，实现了无锁的并发更新。ABA 问题指：线程 A 读取到值 X 后暂停，线程 B 将值改为 Y 再改回 X，A 恢复后 CAS 发现值仍为 X，从而误认为期间无人修改，导致基于“值未变”的逻辑错误。本质是“值相等”不等价于“状态未变”。该机制位于并发编程的底层基石位置，是锁、事务内存、无锁数据结构（栈、队列、哈希表）的构建基础。专业工程师必须掌握它，因为高并发后端和分布式系统中的性能瓶颈往往来自锁竞争，理解 CAS 才能设计正确的无锁算法，并识别 ABA 等隐蔽正确性漏洞。

### 2. 底层原理剖析
CAS 的硬件实现以 x86 的 CMPXCHG 指令为例，配合 LOCK 前缀锁住总线/缓存行，确保比较和替换操作不可被其他核心中断。其语义为：if (*p == expected) { *p = new; return true; } else return false; 循环模式为：do { old = *p; new = f(old); } while (!CAS(p, old, new)); 该循环可能因其他线程持续修改 p 而重试，但系统整体有进度保证（至少一个线程成功）。对比前端已有概念：JavaScript 的单线程事件循环天然避免了共享内存竞争，因此前端工程师通常不需要锁；但 Web Worker 引入共享内存后，Atomics.compareExchange 提供了与后端 CAS 完全一致的底层原语。而 TypeScript 的接口是编译期类型约束，与运行时原子操作无关——这与 Java 的 interface 类似，都是类型契约；CAS 则属于运行时并发控制，两者层次完全不同。底层需注意内存序：CAS 可附带 acquire/release 语义，防止指令重排导致可见性问题。ABA 解决思路：为变量附加版本号（stamp），每次修改递增，CAS 同时比较值和版本号；或在指针场景使用 tagged pointer，将 ABA 视为“值相同但版本不同”。更激进的方法是使用 DCAS（双词 CAS）或 LL/SC（load-link/store-conditional）指令，LL/SC 在写检测时比较地址的引用计数，天然能检测到中间修改。

### 3. 基础代码与实战验证
```text
极简演示（伪代码，基于双字 CAS 或 LL/SC 的真实实现）：
共享结构：value + tag（版本号）
  typedef struct { int value; int tag; } AtomicStampedReference;
单条 CAS 指令同时比较并交换 value 和 tag（需硬件支持双字 CAS）
  bool CAS_WithStamp(AtomicStampedReference* ref, int expectedValue, int expectedTag, int newValue, int newTag) {
      if (ref->value == expectedValue && ref->tag == expectedTag) {
          ref->value = newValue;
          ref->tag = newTag;
          return true;
      }
      return false;
  }
无锁更新循环
  void update(AtomicStampedReference* ref, int (*f)(int)) {
      int v, t, nv;
      do {
          v = ref->value;
          t = ref->tag;
          nv = f(v);
      } while (!CAS_WithStamp(ref, v, t, nv, t + 1));
  }
实战验证：线程1读取到 v=0,t=0；线程2将 v 改为1,t=1，再改为0,t=2；线程1的 CAS 因为 t 从0变2而失败，从而避免误判。循环重试后读取到 t=2，继续操作。
```

### 4. 常见误区与进阶思考
常见误区1：认为 CAS 循环一定比锁快。实际上，在高竞争场景下，CAS 循环可能导致大量重试和缓存行颠簸（cache line bouncing），性能反而下降；且无法保证无等待（wait-free），可能活锁（线程饥饿）。误区2：忽略 ABA 问题，只比较值相等就认为状态未变。例如无锁栈的 pop 操作若仅比较头指针，可能因 ABA 导致把已被复用的节点弹出，产生内存错误。正确的做法是使用版本号或标记，或使用 LL/SC 等可检测中间修改的指令。
思考题：在一个 64 位系统上，如果无锁栈的节点指针恰好能被操作系统复用为相同的地址，ABA 问题是否一定发生？如何仅用单字 CAS（无法同时携带版本号）在栈操作中消除 ABA 的影响？提示：考虑在节点中存储唯一的 tag，而栈顶指针指向该 tag 而不是裸地址。
