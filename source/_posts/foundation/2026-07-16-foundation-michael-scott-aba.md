---
title: "每日基础技术总结 · 2026-07-16 · 无锁队列 Michael-Scott 与 ABA 问题"
date: 2026-07-16 08:00:00
categories: [技术分享]
tags: ["技术分享", "编程语言底层"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-16 · 无锁队列 Michael-Scott 与 ABA 问题

## 📚 今日主题

> **无锁队列 Michael-Scott 与 ABA 问题**（编程语言底层）

### 1. 核心概念速览
无锁队列 Michael-Scott (MS-queue) 是一种基于原子操作（CAS）实现的多生产者多消费者 FIFO 队列，无需任何锁。其本质是用链表结构配合 CAS 原子地更新头尾指针，保证操作在并发冲突时仍能推进（无锁进展）。它解决锁机制带来的阻塞、死锁、优先级反转、上下文切换开销等问题，适用于高并发低延迟场景。在计算机体系中，它属于并发数据结构的核心部分，是构建无锁内存池、并发任务调度、运行时系统的基础。专业工程师必须掌握，因为无锁编程是理解并发本质、硬件原子性、内存模型（如 C++ memory_order）和分布式一致性的关键环节，也是后端与 AI 系统中高吞吐管线的基石。

### 2. 底层原理剖析
MS-queue 维护一个单向链表，含一个哨兵头节点。head 指向哨兵，tail 指向最后一个节点（初始为哨兵）。入队时，线程循环读取 tail 和 tail->next：若 tail->next 为空，则 CAS 将 tail->next 从 null 改为新节点，成功后 CAS 将 tail 推进到新节点；若 tail->next 非空，说明已有线程插入了节点但尚未推进 tail，当前线程帮助推进 tail。出队时，读取 head 和 head->next：若 head->next 为空则队列空；否则 CAS 将 head 推进到 head->next，并返回其值。这里的关键是 CAS 操作必须能够检测 ABA 问题。ABA 问题是指：线程 A 读取指针 X，随后线程 B 将 X 修改为 Y 再改回 X，A 再次 CAS 时看到值仍为 X，以为没变，但实际上链表结构已变。MS 队列的解决方式是使用带计数器的指针（如 Ref 结构），在每次指针更新时同时递增计数器，CAS 比较 (ptr, count) 整体，从而识别 ABA。与前端已有的概念对比：前端 JS 基于单线程事件循环，通过异步非阻塞规避数据竞争，而无锁队列面向真实多线程并行，通过硬件原子指令协调共享内存访问；JS 的 SharedArrayBuffer 和 Atomics 提供了底层原子操作，但很少需要构建无锁队列。另外，TypeScript 的接口是编译期类型契约，与运行时的无锁数据结构属于不同抽象层次，不能混淆。

### 3. 基础代码与实战验证
```text
// 伪代码：基于双宽CAS的MS-queue
struct Node { int val; Node* next; };
struct Ref { Node* ptr; uint64_t count; }; // 带计数器的指针

void enqueue(int v) {
  Node* n = new Node(v);
  while (true) {
    Ref t = tail;                 // 读取尾指针
    Node* next = t.ptr->next;     // 读取尾节点next
    if (next == nullptr) {
      // 尝试将新节点链接到尾节点
      if (CAS(&t.ptr->next, nullptr, n)) {
        // 成功入链，尝试推进tail（失败也无妨）
        CAS(&tail, t, Ref(n, t.count+1));
        return;
      }
    } else {
      // 有线程已入链但未推进tail，帮助推进
      CAS(&tail, t, Ref(next, t.count+1));
    }
  }
}

int dequeue() {
  while (true) {
    Ref h = head;                 // 读头指针
    Node* next = h.ptr->next;     // 哨兵的下一个
    if (next == nullptr) return EMPTY; // 空队列
    if (CAS(&head, h, Ref(next, h.count+1))) {
      return next->val;           // 返回数据，哨兵前移
    }
  }
}
```

### 4. 常见误区与进阶思考
误区一：认为 CAS 比较指针值即可保证安全，忽略 ABA。实际上，CAS 只保证“比较并交换”的原子性，不能区分值是否曾经历“改回”过程。在 MS-queue 中，若指针被回收再复用，可能发生 ABA 导致误判，必须为指针附加版本号（如计数器）或使用危险指针等机制。误区二：认为无锁队列总是比有锁队列性能好。无锁算法在竞争激烈时可能产生大量重试和缓存一致性开销，甚至不如自旋锁；且需要额外处理内存回收，工程复杂度高。思考题：在 MS-queue 的 dequeue 中，如果仅使用普通 CAS 比较 head 指针（无计数器），但假设系统内存分配器保证已释放节点地址永远不会被重新分配（即地址唯一），那么是否仍会发生 ABA 问题？请从链表结构变化角度分析。这需要你理解 ABA 的本质是“值相同但状态不同”，而非单纯的地址复用。
