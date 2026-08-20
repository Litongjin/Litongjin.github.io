---
title: "每日基础技术总结 · 2026-07-30 · V8 垃圾回收：新生代 Scavenge 与对象晋升"
date: 2026-07-30 08:00:00
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-30 · V8 垃圾回收：新生代 Scavenge 与对象晋升

## 📚 今日主题

> **V8 垃圾回收：新生代 Scavenge 与对象晋升**（前端底层与计算机基础）

### 1. 核心概念速览
V8 的堆内存按代际划分为新生代（New Space）与老生代（Old Space）。新生代采用半空间（Semi-space）复制算法，即 Scavenge：两块等大的空间，From 空间负责分配对象，To 空间作为回收目标。当 From 空间耗尽时，标记存活对象并复制到 To 空间，随后 From/To 角色互换，原 From 被整体清空。对象晋升（Promotion）指经过多次 Scavenge 后仍存活的对象，或大小超过指定阈值的对象，被移入老生代。该机制解决的核心问题是：大多数对象生命周期极短，复制存活对象比标记-清除整个新生代更高效，且无内存碎片。它在 V8 整体 GC 体系中承担高频的、面向短命对象的回收任务，与老生代的 Mark-Compact 形成层次化配合。专业工程师必须掌握它，因为前端性能瓶颈往往来自 GC 停顿与内存膨胀，理解代际划分和晋升条件才能解释为何某些代码模式（如长期持有临时变量、闭包误用）会导致老生代堆积，从而从底层优化内存与帧率。

### 2. 底层原理剖析
Scavenge 的本质是空间换时间：将新生代均分为两个半区，任何时刻仅一个半区处于激活态（From）。分配指针（bump pointer）在 From 中线性移动，分配成本仅为指针递增。GC 触发时，从根集（栈、寄存器、老生代中的跨代引用）出发标记 From 中的存活对象，按序复制到 To 空间，同时更新对象内部引用及老生代对新生代对象的引用（写屏障记录了这些跨代指针）。复制完成后，From 整体变为废弃空间，To 成为新的 From。晋升机制：对象经历一次 Scavenge 后年龄（age）加 1，若年龄达到阈值（通常 2 次，受动态启发式调整），或对象大小超过 To 空间的一定比例（如 25%，防止复制开销过大），则直接放入老生代。晋升时需保证老生代空间足够，否则触发老生代 GC。伪代码：

function scavenge() {
  fromSpace.foreach(obj => {
    if (isLive(obj)) {
      if (obj.age >= THRESHOLD || obj.size > TO_SPACE_PERCENT * toSpace.size) {
        promote(obj); // 拷贝到老生代，更新老生代中指向它的引用
      } else {
        obj.age++;
        copyTo(toSpace, obj);
      }
    }
  });
  swap(fromSpace, toSpace);
  toSpace.wipe(); // 逻辑清空
}

与前端已有概念的对比：它类似 Java 的接口与 TypeScript 的接口——名称相似但层次不同。Java 接口是运行时类型系统的契约，参与编译期检查与动态分派；TS 接口仅是编译期结构约束，编译后消失。同样，V8 的 Scavenge 和传统标记-清除都叫 GC 算法，但 Scavenge 依赖代际假设（弱分代假说），只处理新生代；而标记-清除处理整个堆，且不移动对象（产生碎片）。更本质的对比：Scavenge 是复制收集器，牺牲一半空间换取分配效率与无碎片；老生代 Mark-Compact 是标记-整理收集器，牺牲暂停时间换取空间利用率。这与 JS 中 const 与 Object.freeze 的关系类似：const 是编译期不可赋值，Object.freeze 是运行时不可变，不同层面解决不同问题。

### 3. 基础代码与实战验证
```text
// 以下为可运行的 Node.js 实验，观察对象晋升对内存分布的影响。
// 使用 --max-old-space-size 限制老生代，配合 --trace-gc 观察 GC 日志。

// 步骤 1：验证新生代对象快速死亡，不进入老生代
function shortLived() {
  for (let i = 0; i < 1e6; i++) {
    const temp = { id: i }; // 每次循环创建新对象，循环结束即失去引用
  }
}
// 运行：node --trace-gc --max-old-space-size=16 experiment.js
// 观察：大量 Scavenge，但老生代增长极少。

// 步骤 2：验证长期存活对象晋升
let longLived = [];
function promoteDemo() {
  for (let i = 0; i < 10000; i++) {
    longLived.push({ id: i, payload: new Array(1000).fill(i) }); // 大对象，首次晋升
  }
}
// 运行后观察日志中 “Promotion” 相关记录，对象从 new_space 移入 old_space。

// 步骤 3：使用 WeakRef 观察 GC 后对象是否被回收（需要 --expose-gc）
global.gc(); // 强制 GC
const ref = new WeakRef({ data: 1 });
console.log(ref.deref()); // { data: 1 }
global.gc();
console.log(ref.deref()); // undefined，说明无强引用时对象被回收

// 底层解释：
// - 新生代分配是 bump pointer，创建 1e6 个临时对象只会推进分配指针，不会触发老生代 GC。
// - 大对象（超过 To 空间 25%）直接晋升，因为复制大对象成本高，且其寿命往往长。
// - WeakRef 不阻止 GC，但 deref() 在对象被回收后返回 undefined，验证了 GC 确实释放了内存。
// 生产环境中应避免长期持有大对象，防止晋升后触发频繁 Mark-Compact 导致停顿。
```

### 4. 常见误区与进阶思考
误区 1：认为对象一旦不再被引用就会立即被 GC 回收。实际上 V8 的 GC 是延迟的、分代的，一个失去引用的新生代对象要等到下一次 Scavenge 才会被回收；若它已经晋升到老生代，则要等老生代 GC。因此内存峰值可能高于预期，尤其在高频创建临时对象的场景（如 React 渲染中的内联对象）。误区 2：认为闭包必然导致内存泄漏。闭包只延长其捕获变量的生命周期，如果闭包本身存活，捕获对象无法被回收，这是语言语义，并非 GC 缺陷；但若闭包被全局引用且捕获了大型对象，则对象会晋升到老生代且长期驻留，造成内存膨胀。

思考题：V8 在 Scavenge 时需要更新老生代中指向新生代对象的引用。如果不使用写屏障（write barrier）记录这些跨代引用，而是每次 GC 时全量扫描老生代，会对性能产生什么影响？请从分配延迟、GC 暂停时间、缓存局部性三个维度分析，并说明为什么 V8 选择在写入时记录（记忆集）而非扫描时发现。
