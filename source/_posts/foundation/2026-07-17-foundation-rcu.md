---
title: "每日基础技术总结 · 2026-07-17 · RCU 机制与宽限期"
date: 2026-07-17 08:00:00
categories: [技术分享]
tags: ["技术分享", "编程语言底层"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-17 · RCU 机制与宽限期

## 📚 今日主题

> **RCU 机制与宽限期**（编程语言底层）

### 1. 核心概念速览
RCU（Read-Copy Update）是一种并发同步机制，核心思想是将读取与更新解耦，使读者在几乎无开销的情况下并发访问共享数据，而写者通过“复制—修改—发布—宽限期回收”四步完成安全更新。它解决的核心问题是：多读者与单写者并发场景下，如何让读者不阻塞、不加锁，同时保证旧数据在不再被任何读者引用后安全释放。本质是一种延迟回收机制——不立即释放旧版本，而是等待所有可能持有旧引用的读者离开临界区。RCU在计算机体系中属于操作系统内核级同步原语，广泛应用于Linux内核的链表、路由表、文件系统等读多写少的场景；在AI体系中，它同样可用于并发推理服务中的模型热更新。专业工程师必须掌握RCU，因为它揭示了并发编程中“读写不对称”的底层范式，是理解无锁数据结构和内存屏障的基石。

### 2. 底层原理剖析
RCU的底层运行机制可分为读者侧和写者侧。读者侧：进入临界区时调用rcu_read_lock()，退出时调用rcu_read_unlock()。这两个操作在非抢占内核中通常只是关闭/开启内核抢占，不涉及原子指令和内存屏障，因此开销极低。读者在临界区内通过rcu_dereference()读取共享指针，该宏内含必要的编译器/CPU内存屏障，保证读取到的指针是发布时完整的。写者侧：第一步，复制旧数据到新内存（copy）；第二步，修改新副本（update）；第三步，通过rcu_assign_pointer()发布新指针，使读者可见，该宏内含有store-release语义，确保新指针的发布不会早于副本的修改完成（发布，publish）；第四步，调用synchronize_rcu()等待宽限期（grace period）结束。宽限期的定义为：在发布新指针之前已经开始的所有读者临界区全部退出。因为发布之后才进入的读者必然通过新指针访问新数据，不再持有旧引用。等待结束后，写者才能安全释放旧数据。\n\n流程伪代码：\n\n// 写者更新\nnew = alloc_and_copy(old);\nnew->field = new_value;\nrcu_assign_pointer(global_ptr, new); // store-release\nsynchronize_rcu(); // 等待宽限期\nkfree(old);\n\n// 读者\nrcu_read_lock();\np = rcu_dereference(global_ptr); // load-acquire\nuse(p);\nrcu_read_unlock();\n\n与前端已有概念的对比：前端中的“双缓冲渲染”与RCU有相似之处——双缓冲允许读取者（显示器）在后台缓冲区被修改时继续读取前台缓冲区，通过切换指针完成更新，读者无需等待。不同点在于：双缓冲通常只有两个固定缓冲区，且切换由硬件垂直同步信号驱动，不存在显式的宽限期；RCU则支持任意数量的读者并发读取旧版本，且通过synchronize_rcu精确等待所有旧读者退出，保证内存回收安全。另外，前端状态管理中的不可变数据更新（如Redux reducer返回新对象）也类似RCU的Copy阶段，但缺少宽限期这一关键回收机制，通常依赖垃圾回收而非显式同步。

### 3. 基础代码与实战验证
```text
以下为Linux内核风格RCU读写伪代码，展示核心机制。\n\n// 共享数据指针\nstruct foo *g_ptr;\n\n// 读者：无锁读取\nvoid reader(void)\n{\n    struct foo *p;\n\n    rcu_read_lock();            // 进入读端临界区（禁止抢占，保证临界区不被调度）\n    p = rcu_dereference(g_ptr); // 读取指针，内含内存屏障，确保读取顺序正确\n    // 此时p可能指向旧数据或新数据，取决于是否在写者发布之后\n    do_something_with(p);       // 安全使用p指向的数据\n    rcu_read_unlock();          // 退出临界区，允许抢占\n}\n\n// 写者：复制-修改-发布-等待-回收\nvoid writer(void)\n{\n    struct foo *old, *new;\n\n    old = rcu_dereference(g_ptr);   // 获取当前指针\n    new = kmalloc(sizeof(*new));    // 分配新内存\n    *new = *old;                    // 复制旧数据（Copy）\n    new->field = updated_value;     // 修改新副本（Update）\n    rcu_assign_pointer(g_ptr, new); // 发布新指针，store-release，读者从此可见新数据\n\n    synchronize_rcu();              // 阻塞等待宽限期结束，即所有在发布前进入的读者已退出\n\n    kfree(old);                     // 安全释放旧数据，因为不再有读者引用旧指针\n}\n\n注意：synchronize_rcu()在Linux中会触发上下文切换，可能睡眠，因此写者不能在原子上下文调用。实际中可用call_rcu()异步回调替代。
```

### 4. 常见误区与进阶思考
误区1：认为RCU可以完全替代读写锁。实际上，RCU适用于读多写少、写者可以容忍延迟和复制开销的场景；对于写频繁或读者临界区过长的场景，宽限期等待和复制成本会显著降低性能，甚至导致写者饥饿。\n误区2：认为synchronize_rcu()必须等待所有读者（包括宽限期开始之后进入的读者）都退出。正确语义是：只等待在发布新指针之前已经进入的读者退出。因为发布之后进入的读者会通过rcu_dereference读取到新指针，不会访问旧数据，所以无需等待。错误理解会导致过度等待和性能下降。\n\n思考题：如果一个读者在rcu_read_lock()和rcu_read_unlock()之间发生睡眠或阻塞，宽限期会如何？Linux内核为何区分不可睡眠RCU（如synchronize_rcu）与可睡眠RCU（如SRCU）？请从调度器与抢占机制的角度解释其设计取舍。
