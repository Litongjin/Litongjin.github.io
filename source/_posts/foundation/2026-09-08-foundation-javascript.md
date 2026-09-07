---
title: "每日基础技术总结 · 2026-09-08 · JavaScript 垃圾回收机制"
date: 2026-09-08 07:13:39
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-08 · JavaScript 垃圾回收机制

## 📚 今日主题

> **JavaScript 垃圾回收机制**（前端底层与计算机基础）

### 1. 核心概念速览
JavaScript 垃圾回收机制（GC）是 JavaScript 引擎（如 V8）自动管理堆内存生命周期的机制。它的本质是：通过追踪变量的可达性（reachability）确定哪些对象不再被任何根（root：全局对象、当前执行上下文中的局部变量、参数、调用栈上的引用等）引用，并回收其内存。解决的问题是：让开发者无需手工分配和释放堆内存，避免内存泄漏与悬垂指针（dangling pointer）。其机制是：从根节点出发遍历对象图，标记所有可到达对象，并清除不可达对象。在计算机体系中，GC 是运行时（runtime）的核心组件之一，与编译优化、事件循环、异步 I/O 处于同一层。对专业工程师而言，理解 GC 是分析内存泄漏、优化长时间运行的前端应用（如 SPA）和 Node.js 服务的必要条件，也是理解 WeakMap、WeakRef、FinalizationRegistry 以及闭包引用的底层基石。

### 2. 底层原理剖析
V8 采用分代式收集（Generational GC），把堆分为新生代（Young Generation）与老生代（Old Generation）。新生代采用 Scavenger（复制）算法，对象在伊甸园（Eden）和两个幸存空间（Survivor）中轮转；老生代采用标记-清除-整理（Mark-Sweep-Compact）算法。

运行流程：
1. 分配：新对象在新生代分配；新生代空间不足时触发 Minor GC。
2. Minor GC：从栈上的局部变量、全局对象及已注册的根引用出发，用三色标记法遍历对象图，标记所有可达对象；将存活对象复制到另一块幸存空间，并使其年龄 +1；年龄达到阈值（如 2 或 4）的对象晋升到老生代。
3. Major GC：当老生代内存不足或碎片过多时触发。先做标记（Mark），DFS 遍历所有可达对象；再清除（Sweep），将不可达对象所在的内存块加入空闲链表；必要时做整理（Compact），把存活对象移动到内存区域一端，减少碎片化。
4. 渐进式：为避免 Stop-The-World 长停顿，V8 采用增量标记（Incremental Marking），将标记步骤切分成小段穿插在 JS 执行中；同时用并发线程执行清除（Concurrent Sweeping），降低主线程卡顿。

与前端已有知识体系的对比：基本类型的值在栈中，对象在堆中，GC 只处理堆上的对象引用关系。这与 C++/Rust 的显式内存管理不同——JS 没有手动释放堆内存的 API，也没有所有权转移概念。与类型系统的对比：TS 的接口是编译期静态约束，运行时不存在；GC 的可达性分析完全不关心变量的静态类型，只关心对象间的引用图。这一点类似 Java 的接口与 TS 的接口在运行时都会被擦除（erasure），它们解决的是开发期的抽象约束，而 GC 解决的是运行期的内存复用，两种机制在运行时是解耦的。

### 3. 基础代码与实战验证
```text
// 验证 GC 的可达性回收——使用 WeakMap 观察弱引用
const weakMap = new WeakMap();
let key = {};
weakMap.set(key, new Array(1024 * 1024).fill(1)); // 键时一个对象，值是一个大数组
// 此时 key 变量强引用该对象，WeakMap 对 key 是弱引用，不阻止 GC
key = null; // 解除唯一强引用，对象变为不可达
// 在支持 global.gc() 的环境（Node 的 --expose-gc）下强制 GC，可验证对象被回收。
// 若用 Map 代替 WeakMap，则 key 被 Map 强引用，即使 key 变量为 null，也无法回收。

// 验证 FinalizationRegistry：对象被回收时触发回调
const registry = new FinalizationRegistry(held => {
  console.log('finalized:', held);
});
let obj = { data: new Uint8Array(1024) };
registry.register(obj, 'obj-key');
obj = null; // 等变量变为不可达，GC 运行后回调可能在异步时机触发
// 注意：FinalizationRegistry 不保证立即执行，也不保证一定执行，仅用于诊断。
```

### 4. 常见误区与进阶思考
误区 1：将变量置为 null 或 undefined 会立刻释放内存。事实是，置空只是让该对象从当前根不可达，GC 是否回收、何时回收由引擎策略决定，而且对象可能被事件监听器、闭包、缓存等隐藏引用持有。专业工程师应通过记录/快照（如 DevTools Heap Snapshot）查找实际引用路径，而不是盲目置空。

误区 2：WeakMap 能解决所有内存泄漏。事实是：WeakMap 的键必须是对象，且只能防止键对象本身被回收；如果值里引用了键，形成循环引用，现代追踪式 GC 可以处理；但 WeakMap 不可枚举，也无法遍历，若用它实现缓存会失去批量操作能力。真正要掌握的是：为什么 Map 的强引用会导致泄漏，以及 WeakMap 的“弱”只作用于键，不作用于值。

思考题：一个业务对象被某个事件监听器捕获，而这个监听器又注册在 Window 上，但业务代码已经将所有对该业务对象的引用置空。请问该对象是否可被回收？请从 GC 根路径（root path）分析：Window 是根，事件监听器是根可达对象，监听器可能通过闭包持有业务对象，所以业务对象仍可被根到达，因此不可回收。这与“不可访问”并不等价——只有从根出发不可达才会被回收，请思考闭包环境如何形成这样的根路径。
