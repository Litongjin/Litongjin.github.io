---
title: "每日基础技术总结 · 2026-08-20 · V8 新生代 Scavenger 与老生代 Mark-Compact 回收"
date: 2026-08-20 08:00:00
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-08-20 · V8 新生代 Scavenger 与老生代 Mark-Compact 回收

## 📚 今日主题

> **V8 新生代 Scavenger 与老生代 Mark-Compact 回收**（前端底层与计算机基础）

### 1. 核心概念速览
V8 的堆内存按对象生存期分为新生代（Young Generation）与老生代（Old Generation）。新生代采用半空间（Semi-space）复制算法 Scavenger（具体为 Cheney 算法），老生代采用 Mark-Compact（标记-压缩）回收器。本质是：新生代利用『对象朝生夕死』的统计规律，通过复制存活对象到空闲半空间，一次性回收整个废弃半空间，以空间换时间，避免碎片化；老生代则因对象存活率高、复制代价大，改用标记-清理+压缩（Mark-Sweep-Compact）就地回收，通过移动对象消除碎片。这套分代机制解决的核心问题是：在保证 GC 停顿可控的前提下，最大化吞吐量并最小化内存碎片。它在整个计算机体系中属于自动内存管理（Tracing GC）的分治策略——将堆按对象年龄分而治之，是 V8、JVM（HotSpot）、Go（部分版本）等主流运行时共同采用的工程范式。专业工程师必须掌握它，因为前端性能优化（尤其长列表、动画、WebGL 场景）的卡顿根源常是 GC 停顿，理解分代与回收算法才能正确解读 DevTools 的 Memory 面板、优化对象分配模式，并为深入理解 JVM 或实现语言运行时打下基础。

### 2. 底层原理剖析
新生代 Scavenger 机制：堆分为两个等大的半空间（From-Space 与 To-Space），新对象总是分配在 From-Space。当 From-Space 耗尽时触发回收：从 GC Roots（栈、全局、寄存器）出发遍历对象图，将存活对象按顺序复制到 To-Space，并更新所有引用指针；复制完成后，From-Space 整体视为垃圾，直接清空，然后交换 From/To 角色。由于复制是顺序写入，存活对象在 To-Space 中紧密排列，天然无碎片。对象每经历一次 Scavenger 且存活，年龄（age）加一；当年龄超过阈值（如 15，或对象大小超过新生代最大块，如 8KB 的大对象直接进老生代）则晋升（promote）到老生代。Scavenger 的停顿时间与存活对象数量成正比，而非总分配量，因此对短命对象为主的年轻代极为高效。

老生代 Mark-Compact 机制：老生代空间不再采用复制，因为存活率高，复制代价大。回收分三阶段：1）Mark：从 GC Roots 遍历对象图，标记所有可达对象（用三色标记法实现，白=未访问，灰=已访问但引用未处理，黑=已访问且引用已处理），此阶段会记录每个对象的标记位。2）Sweep：线性扫描堆，将未标记（白色）对象的内存块加入空闲链表，此时会产生碎片。3）Compact：为了避免碎片导致的大对象分配失败，V8 会将存活对象向堆的一端移动，压缩空间，然后更新所有引用地址。Mark-Compact 是 Full GC 的核心，其停顿时间与堆大小及存活对象总量相关，远高于 Scavenger。V8 还引入了增量标记、并发标记、惰性清理等优化，将长时间停顿拆解到多个任务中，但算法本质仍是标记-压缩。

与前端已有概念的对比：这类似于『接口』在不同语言中的语义差异。Java 的接口是编译期契约，强调类型结构约束；TypeScript 的接口是结构类型系统下的鸭子类型，运行时并不存在。类比到 GC：Scavenger 与 Mark-Compact 是两种不同的『内存管理契约』，但它们的本质都是追踪式回收，区别在于对对象存活率假设的取舍。Java 的接口偏向行为规范，TS 接口偏向形状匹配；Scavenger 偏向年轻对象的快速复制，Mark-Compact 偏向老对象的就地整理。理解这种异同，能帮助你从抽象机制层面把握不同运行时的设计哲学。

### 3. 基础代码与实战验证
由于 GC 算法是运行时内部机制，无法用纯 JavaScript 直接调用，但可以用伪代码精确描述核心流程。以下为 Scavenger 的简化实现（伪代码）：

```
// 半空间结构
class SemiSpace {
  base; // 起始地址
  capacity; // 大小
  top; // 分配指针
}

// 对象头包含：map(类型)、age、mark、forwarding_addr
function scavenge(fromSpace, toSpace, roots) {
  // 复制根对象到 toSpace
  for (root in roots) {
    root = copyIfNeeded(root, fromSpace, toSpace);
  }
  // 广度优先遍历已复制对象，更新其字段引用
  scanPtr = toSpace.base;
  while (scanPtr < toSpace.top) {
    obj = scanPtr;
    for (field in obj.fields) {
      field = copyIfNeeded(field, fromSpace, toSpace);
    }
    scanPtr += obj.size;
  }
  // 交换半空间
  swap(fromSpace, toSpace);
  // toSpace 清零，下次分配使用
  toSpace.top = toSpace.base;
}

function copyIfNeeded(obj, fromSpace, toSpace) {
  if (obj.forwarding_addr == null) {
    // 复制对象到 toSpace，并设置转发指针
    newAddr = toSpace.top;
    memcpy(newAddr, obj, obj.size);
    obj.forwarding_addr = newAddr;
    newAddr.age++;
    if (newAddr.age > MAX_AGE) {
      promote(newAddr); // 晋升到老生代
    }
    return newAddr;
  } else {
    return obj.forwarding_addr; // 已复制，直接返回新地址
  }
}
```

Mark-Compact 的三阶段伪代码：

```
function markCompact(roots) {
  // Mark: 三色标记
  for (root in roots) mark(root);
  // Sweep: 回收未标记对象
  sweep();
  // Compact: 移动存活对象，更新引用
  compact();
}

function mark(obj) {
  if (obj.color == WHITE) {
    obj.color = GREY;
    for (field in obj.fields) mark(field);
    obj.color = BLACK;
  }
}
```

关键行注释：`obj.forwarding_addr` 是 Scavenger 的转发指针，用于记录对象被复制后的新位置，这样既能避免重复复制，又能在后续字段更新时找到新地址；`age++` 和 `promote` 实现了对象晋升机制。Mark 阶段的 `GREY` 状态确保对象图的遍历不会遗漏循环引用。实际 V8 实现中，新生代对象分配是 bump-pointer 分配（仅移动 top 指针），老生代使用空闲链表分配，这些细节都服务于减少 GC 停顿。

### 4. 常见误区与进阶思考
误区 1：认为 GC 停顿只取决于堆大小，或认为 Scavenger 总是比 Mark-Compact 快。实际上，Scavenger 的耗时与存活对象数量成正比，而不是分配量；当新生代中存在大量长期存活对象（如缓存、闭包引用的大对象）时，反复复制这些对象会导致 Scavenger 性能下降，甚至触发过早晋升。真正瓶颈是存活率，而非堆容量。误区 2：混淆『可达性』与『存活时间』。Mark-Compact 的标记阶段只判定可达性，而可达对象可能在 GC 后立即变成垃圾；分代假设基于统计概率，不是绝对正确。因此，一个长期被全局变量引用的短命对象（如挂在 window 上的临时数组）会直接导致它晋升到老生代，造成老生代空间膨胀，最终 Full GC 频繁触发。

思考题：在 Scavenger 复制过程中，如果从 From-Space 复制对象到 To-Space 时，对象内某个字段指向的是 From-Space 中尚未被复制的对象，那么该字段的更新时机是什么？如果在复制完所有对象后再统一更新引用，与复制时立即更新相比，各自的空间和时间复杂度如何？请基于 Cheney 算法的广度优先扫描顺序推导，并解释为什么 V8 的 Scavenger 采用『复制时立即更新』而非『事后统一修正』，这背后对缓存局部性和分配顺序有什么影响？
