---
title: "每日基础技术总结 · 2026-09-16 · JavaScript 垃圾回收机制"
date: 2026-09-16 07:02:04
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-16 · JavaScript 垃圾回收机制

## 📚 今日主题

> **JavaScript 垃圾回收机制**（前端底层与计算机基础）

### 1. 核心概念速览
核心概念速览：
- 定义：JavaScript 垃圾回收（Garbage Collection，GC）是 JS 引擎自动管理堆内存的机制，负责识别并回收不再被程序可达引用的对象，释放其占用的内存，避免手动 free/delete 带来的悬垂指针和泄漏。
- 本质：以根集合（globalThis、当前执行上下文栈、闭包环境、DOM/事件等宿主根）为起点，沿引用图做可达性分析；不可达对象即垃圾。
- 解决：堆内存有限，对象生命周期动态，闭包/事件回调/DOM 引用会产生复杂引用图。GC 将内存分配与释放解耦，程序员只管理可达性。
- 机制：V8 采用分代式 GC：新生代 Scavenger（Cheney 复制算法），老生代 Mark-Sweep/Mark-Compact；并发标记、增量标记、并行清扫、空闲 GC；写屏障维护记忆集。
- 位置：位于语言运行时/引擎层，连接内存模型、对象模型、执行上下文、闭包、DOM/C++ 绑定。前端性能、内存泄漏排查、Node 服务稳定性、AI/WebGPU 大内存数据处理都依赖它。
- 为什么必须掌握：不懂可达性会导致闭包、全局缓存、监听器、定时器、脱离 DOM 的引用泄漏；不懂分代与停顿会误判 Performance 面板、无法解释内存增长与 GC 毛刺。

### 2. 底层原理剖析
底层原理剖析：
1. 根集合与可达性：
根包括全局对象、调用栈中的局部变量/参数、闭包 captured 环境、活跃的宿主对象（DOM 节点、事件监听器、定时器、Promise 反应等）。从根出发对对象图做图遍历，访问到的对象为 live，未访问为 garbage。

2. V8 堆分代：
- 新生代（Young Generation / New Space）：小、分 from/to 两个 semispace，存放短命对象。
- 老生代（Old Generation / Old Space）：大、存放多次存活对象和大对象。
- 大对象空间、代码空间、Map 空间等辅助。

3. 新生代 Scavenger：
采用 Cheney 复制算法。伪代码：
scan = from.start
while scan < from.top:
  obj = objectAt(scan)
  for child in obj.references:
    if child not in to-space:
      copy child to to-space
      update child pointer
  scan += obj.size
swap(from, to)
优点：吞吐高、无碎片；代价：需要保留一半 semispace，复制成本。对象经历两次 Scavenger 仍存活或 to-space 超阈值则晋升老生代。

4. 老生代 Mark-Sweep / Mark-Compact：
- 标记：从根出发三色标记：白=未访问，灰=已访问待扫描，黑=已扫描。
- 清除：回收白色对象，将空闲块加入 free list。
- 压缩：存活对象移动合并，消除碎片。
- 并发/增量：为降低 stop-the-world，V8 使用并发标记（在 worker 线程标记）、增量标记（主线程切片）、并行清扫、懒清扫。

5. 写屏障与记忆集：
分代收集需要跨代引用（老对象引用新对象）。若只扫描新生代，会漏掉老生代到新生代的边。写屏障在 obj.field = value 时记录老到新引用到 remembered set；新生代 GC 将 remembered set 作为额外根。

6. 弱引用与终结器：
WeakMap/WeakSet/WeakRef 不阻止键/对象被回收；FinalizationRegistry 在回收后异步执行清理，但不保证时机。

对比前端已有知识：
- 与 Java GC：思想同为可达性分析+分代，但 Java 有强/软/弱/虚引用、显式 System.gc() 建议、finalize 风险；JS 无显式释放，WeakMap 是主要的弱引用出口，且宿主环境（DOM）参与根集合。
- 与 TS 类型：TS 类型在编译期擦除，不进入运行时对象图；接口/泛型不产生 GC 根，类型系统不管理内存。
- 与闭包：闭包是词法环境被函数引用导致的运行时对象，闭包捕获变量会延长可达链。
- 与栈/堆：栈帧出栈后局部变量根消失；堆对象由 GC 决定。V8 可做逃逸分析，将未逃逸对象标量替换到栈/寄存器，减少堆分配。

### 3. 基础代码与实战验证
```text
基础代码与实战验证：
以下 Node/浏览器控制台均可运行，用 WeakRef 和 FinalizationRegistry 观察可达性对回收的影响。注意 GC 时机不确定，需配合 --expose-gc（Node）或 devtools 手动 GC。

// 1. 强引用保持对象可达
let strong = { id: 1 };
// strong 是全局/模块作用域变量，属于根可达路径；只要 strong 不置空，{id:1} 不会被回收。

// 2. WeakRef 不阻止回收
let weak = new WeakRef({ id: 2 });
// WeakRef 持有弱引用：对象若只被 weak 引用，GC 可回收；deref() 返回对象或 undefined。
console.log(weak.deref()?.id); // 2

// 3. FinalizationRegistry 观察回收后的清理回调
let registry = new FinalizationRegistry((held) => {
  // 回调在对象被 GC 回收后异步执行；held 是注册时传入的附属值，不是被回收对象本身。
  console.log('collected:', held);
});

let target = { id: 3 };
registry.register(target, 'target-3');
// register 不创建强引用？注意：registry 对 target 是弱引用，但 target 仍被变量 target 强引用。

// 4. 断开强引用，制造不可达
strong = null;
target = null;
// 此时 {id:1} 和 {id:3} 不再从根可达；实际回收需等待引擎触发 GC。

// 5. Node 中可手动触发 GC 加速验证：node --expose-gc gc-demo.js
if (global.gc) {
  global.gc(); // 请求一次完整 GC；生产代码不应依赖此 API，仅用于验证可达性。
}
setTimeout(() => {
  console.log('weak after gc:', weak.deref()); // 通常为 undefined，表示已被回收。
}, 100);

// 6. 闭包泄漏对照
function makeLeak() {
  const big = new Array(1e6).fill('x');
  return function inner() {
    // inner 的 [[Environment]] 引用 big 所在词法环境；只要 inner 可达，big 就可达。
    return big.length;
  };
}
const leak = makeLeak(); // big 无法回收
// leak = null; // 解除 inner 可达后，big 才进入可回收候选。

关键验证点：
- WeakRef.deref() 在 GC 后可能变为 undefined，证明可达性决定回收。
- FinalizationRegistry 回调不代表立即回收，时机由引擎调度。
- 闭包捕获的环境是运行时引用边，不是编译期类型信息。
```

### 4. 常见误区与进阶思考
常见误区与进阶思考：
误区1：认为引用计数是 JS 的主要 GC 算法，或认为循环引用一定泄漏。现代 V8 使用可达性分析，循环引用若整体不可达会被一起回收；引用计数只是某些实现/局部机制，不是 V8 主策略。
误区2：认为 WeakMap/WeakRef 会立即释放或 FinalizationRegistry 会确定性执行。弱引用只表示不阻止回收，回收时机由 GC 策略决定；FinalizationRegistry 回调异步且可能永不执行，不能用于关键资源释放。
误区3：把 delete obj.prop 当作释放内存。delete 只删除对象属性，可能改变对象形状并导致去优化，不保证释放底层内存；真正释放取决于对象可达性和 GC。
误区4：只看堆快照就断定泄漏。快照中的 retained size 与支配树才是关键；大量灰色节点可能是缓存或引擎内部结构，需要对比多次 GC 后的 retained size。

进阶思考题：
V8 分代 GC 中，如果老生代对象引用了新生代对象，只扫描新生代根集合会漏掉该新生代对象。请从写屏障、记忆集和三色标记不变式角度解释：V8 如何在并发标记期间保证不会把仍存活对象误标为白色？并说明为什么“强引用链存在”并不等价于“对象一定存活到程序下一次使用”，需要从 GC 根集合的精确性、优化编译器逃逸分析/标量替换、以及宿主对象生命周期三个层面回答。
