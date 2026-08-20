---
title: "每日基础技术总结 · 2026-02-21 · V8 新生代 Scavenger 与老生代 Mark-Compact 回收"
date: 2026-02-21 08:00:00
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-02-21 · V8 新生代 Scavenger 与老生代 Mark-Compact 回收

## 📚 今日主题

> **V8 新生代 Scavenger 与老生代 Mark-Compact 回收**（前端底层与计算机基础）

### 1. 核心概念速览
V8 将托管堆划分为新生代与老生代两个代际，基于分代假设（大部分对象朝生夕灭）采用不同回收策略。新生代使用 Scavenger（半空间复制算法），将新生代均分为 From 与 To 两个等大半区，只对存活对象做搬迁式复制，回收后 From/To 互换；老生代使用 Mark-Compact（标记-压缩），先标记可达对象，再滑动压缩消除碎片，避免复制大对象的高成本。该机制解决的核心问题是 JavaScript 自动内存管理中『如何以低停顿、低碎片、高吞吐地回收不同生命周期对象』。它在整个计算机体系中属于运行时内存管理子系统的具体实现，位于语言虚拟机与操作系统内存管理之间，是理解性能优化、内存泄漏、微任务调度、跨代引用等前端底层行为的基础。专业工程师必须掌握它，因为任何 Node.js 服务的内存表现、长任务卡顿、堆快照分析最终都追溯到 GC 算法与代际策略的交互，而非停留在调参或盲目优化。

### 2. 底层原理剖析
V8 的堆结构在代际层面分为新生代（New Space）与老生代（Old Space），新生代容量远小于老生代，典型为 16~32MB，老生代可达数 GB。

1. 新生代 Scavenger：
- 分配：新生代被划分为 From 半区和 To 半区，新对象一律在 From 中分配，To 始终空闲。当 From 容量耗尽触发 Scavenger。
- 标记存活：从根集（栈、寄存器、全局、老生代中的跨代引用记录表）出发，遍历对象图，标记 From 中的存活对象。
- 复制迁移：将存活对象按原顺序复制到 To 半区，同时更新所有指向被迁移对象的指针。复制过程天然完成压缩，因为对象被紧凑排列在 To 中。
- 角色互换：清空 From，将 From 与 To 互换，新分配继续发生在新的 From。
- 晋升：若对象经历一次 Scavenger 后仍存活，或 To 半区空间占用超过一定阈值（如 25%），则将该对象直接复制到老生代，避免反复复制。
- 本质：复制算法只遍历存活对象，死亡对象被整体丢弃，无标记清理的碎片问题，代价是内存利用率减半，且移动对象需要更新引用。

2. 老生代 Mark-Compact：
- 标记阶段：从根集和新生代中的跨代引用表出发，深度优先遍历对象图，为每个可达对象打上标记位。由于老生代对象量大，V8 采用三色标记法（白灰黑）实现增量标记，配合写屏障（Write Barrier）记录增量标记期间新产生的引用，避免漏标。
- 压缩阶段：标记完成后，将所有存活对象向堆的一端滑动，使存活对象连续排列，并更新所有指向移动后对象的指针。压缩消除了内存碎片，但需要二次遍历或记录移动偏移，开销高于单纯标记-清除。
- V8 会根据碎片率与堆压力选择三种模式：Mark-Sweep（仅清理死亡对象，不移动）、Mark-Compact（移动存活对象）、Mark-Sweep+增量整理（折中）。最终仍会执行压缩，以保证大对象分配成功。
- 写屏障：新生代与老生代之间互相存在指针。新生代 Scavenger 复制时需扫描老生代中的跨代指针，V8 使用存储缓冲区（Store Buffer）记录老生代对象写入新生代指针的事件，避免全堆扫描。老生代增量标记时同样用写屏障维护三色不变式。

3. 与前端已有概念的对比：
- 类似『Java 的接口与 TypeScript 的接口』：前者是运行时类型系统的行为约束（JVM 通过方法表实现动态分派），后者是编译期结构类型检查（擦除后无运行时实体）。类比 GC 中『复制 vs 标记-压缩』：复制算法是运行时物理搬迁，标记压缩是逻辑整理；但本质都是为达成『内存连续可用』这一目标的不同机制。更贴切的是『引用计数 vs 可达性分析』——引用计数是即时、局部的判定（类似 TS 的结构子类型），可达性分析是延迟、全局的判定（类似 Java 接口的运行时多态），两者在循环引用问题上分道扬镳。
- 前端工程师熟悉的事件循环与微任务：微任务队列中的回调可能持有对象引用，导致对象在 GC 根集中存活。GC 的根集包含了当前正在执行的微任务及其闭包环境，这与浏览器渲染的帧生命周期相互作用。

### 3. 基础代码与实战验证
```text
以下为文字化伪代码，精确描述 V8 新生代 Scavenger 与老生代 Mark-Compact 的核心步骤，不依赖具体框架。

// 新生代 Scavenger
function scavenge(newSpace, roots, storeBuffer) {
  // 1. 遍历根集与 storeBuffer 中的跨代指针，找到 From 半区存活对象
  markFrom(roots, storeBuffer);

  // 2. 遍历 From 半区，对每个存活对象复制到 To 半区
  for (obj in fromSpace) {
    if (obj.isMarked && !obj.isForwarded) {
      // 3. 若对象已存活过一次且 To 空间充足，则晋升到老生代
      if (obj.age >= promoteThreshold || toSpace.used > toSpace.capacity * 0.25) {
        newAddr = oldSpace.allocate(obj.size);
      } else {
        newAddr = toSpace.allocate(obj.size);
      }
      // 4. 复制对象内容，并在原对象头部写入转发指针（forwarding pointer）
      copyMemory(newAddr, obj, obj.size);
      obj.forwardingPointer = newAddr;
      obj.age++;
    }
  }

  // 5. 修正所有指向被移动对象的引用：遍历 From 半区，读取转发指针更新
  updateReferences(fromSpace, toSpace, oldSpace);

  // 6. 清空 From 半区，交换 From/To 角色
  fromSpace.clear();
  swap(fromSpace, toSpace);
}

// 老生代 Mark-Compact（简化，使用三色标记）
function markCompact(heap, roots, storeBuffer) {
  // 1. 初始化所有对象为白色
  for (obj in heap.oldSpace) { obj.color = WHITE; }

  // 2. 从根集与 storeBuffer 开始深度优先标记（灰→黑）
  stack = [];
  for (root in roots) stack.push(root);
  for (addr in storeBuffer) stack.push(addr); // 跨代引用中的新生代对象
  while (stack not empty) {
    obj = stack.pop();
    if (obj.color == WHITE) {
      obj.color = GREY;
      // 3. 扫描 obj 的所有字段，若引用白色对象则入栈
      for (field in obj.fields) {
        if (field.color == WHITE) stack.push(field);
      }
      obj.color = BLACK; // 标记完成
    }
  }

  // 4. 若需要压缩：计算每个存活对象的新地址（滑动指针）
  if (heap.fragmentation > threshold) {
    newOffset = heap.oldSpace.start;
    for (obj in heap.oldSpace) {
      if (obj.color == BLACK) {
        obj.newAddress = newOffset;
        newOffset += obj.size;
      }
    }
    // 5. 第二次遍历更新所有指向存活对象的指针
    updateAllReferencesToNewAddresses(heap);
    // 6. 移动对象内容（批量 memmove）
    for (obj in heap.oldSpace) {
      if (obj.color == BLACK) moveObject(obj);
    }
  } else {
    // 仅清除白色对象
    sweep(heap.oldSpace);
  }
}

// 写屏障示例：老生代 obj 引用新生代 newObj 时，记录到 storeBuffer
function writeBarrier(obj, field, newObj) {
  if (obj in oldSpace && newObj in newSpace) {
    storeBuffer.add(obj);
  }
  obj.field = newObj;
}

// 验证方法：在 Node.js 中通过 --trace-gc 观察 Scavenger 与 Mark-Compact 的触发
// 命令行：node --trace-gc --max-old-space-size=64 test.js
// 代码：
// const arr = [];
// for (let i = 0; i < 1e7; i++) {
//   arr.push({ a: i, b: new Array(10).fill(i) }); // 创建大量短生命周期对象
//   if (i % 1e5 === 0) arr.length = 0; // 清空数组，让对象不可达
// }
// 观察输出中的 'Scavenge' 与 'Mark-Compact' 事件，对应代际回收发生时机。
```

### 4. 常见误区与进阶思考
误区一：认为 Scavenger 和 Mark-Compact 是互斥的两种 GC 算法，V8 只选择其一。实际上二者是代际策略的协同组件：新生代频繁 Scavenger，老生代才使用 Mark-Compact（或 Mark-Sweep），且对象晋升、跨代引用屏障、增量标记共同组成完整回收流程。将二者割裂理解会导致误判性能瓶颈，例如把大对象分配失败归因于新生代频繁 GC，实际是老生代碎片触发压缩。

误区二：认为 Mark-Compact 的压缩阶段会移动所有对象，因此每次 GC 都会导致地址变化。实际 V8 优先使用 Mark-Sweep（仅清理不移动），只有碎片率过高或分配失败时才触发 Mark-Compact 的压缩。移动对象会产生巨大的引用更新开销，所以 V8 有专门的碎片整理策略（如增量整理、仅整理部分页），并非每次老生代回收都全量压缩。

思考题：给定一个对象 A 被老生代对象 B 引用，同时 A 被根引用。在新生代 Scavenger 执行过程中，A 位于新生代 From 半区。如果没有 Store Buffer，GC 如何保证 B 对 A 的引用在 A 迁移后仍然正确？请分析引入 Store Buffer 后，GC 是否需要再次扫描 B 的全部字段来更新指针，以及如果 B 在 Mark-Compact 中被移动，Store Buffer 中的记录是否会失效？
