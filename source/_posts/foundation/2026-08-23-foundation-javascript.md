---
title: "每日基础技术总结 · 2026-08-23 · JavaScript 垃圾回收机制"
date: 2026-08-23 06:55:52
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-23 · JavaScript 垃圾回收机制

## 📚 今日主题

> **JavaScript 垃圾回收机制**（前端底层与计算机基础）

### 1. 核心概念速览
JavaScript 垃圾回收（Garbage Collection, GC）是一种自动内存管理机制，其本质是运行时系统通过追踪对象引用的可达性（Reachability），自动识别并回收无法再被程序访问的内存块。它解决的问题是：在无显式内存分配/释放的语义下，保证堆内存空间的高效复用，同时避免悬垂指针和双重释放等手动管理错误。该机制内嵌于 JavaScript 引擎（如 V8、SpiderMonkey）中，属于程序语言运行时（Runtime）的核心组件，与编译器、解释器、操作系统虚拟内存管理共同构成计算机系统执行模型的一环。专业工程师必须掌握它，因为前端应用的长期运行（如 SPA、Node.js 服务）高度依赖 GC 的正确性和性能；不理解可达性和引用类型，就无法诊断内存泄漏、优化动画性能、设计合理的数据缓存策略，更无法深入理解框架（如 React、Vue）的响应式依赖管理为何使用 WeakMap 等弱引用结构。

### 2. 底层原理剖析
JavaScript 的 GC 核心基于可达性（Reachability）判定。从一组根（Roots）出发——根包括全局对象（window/globalThis）、当前调用栈中的局部变量和参数、以及活跃的微任务和宏任务环境中的引用——沿着引用边遍历对象图，能到达的对象为存活对象，其余均视为可回收。现代引擎（以 V8 为例）采用分代式堆管理：新生代（Young Generation）存放短命对象，使用半空间复制（Scavenger，基于 Cheney 算法）进行高频率小规模回收；老生代（Old Generation）存放经过多次存活提升的对象，使用标记-清除（Mark-Sweep）和标记-整理（Mark-Compact）进行低频大规模回收。标记阶段采用三色标记法：白色表示未访问，灰色表示已访问但未扫描其引用，黑色表示已访问且已扫描。初始所有对象为白色，从根开始将引用的对象置灰并压入栈，随后不断弹出灰色对象，将其引用的白色对象置灰，自身变黑，直到栈空。清除阶段直接回收白色对象，若内存碎片严重则执行整理（移动对象更新引用）。为避免全停顿（Stop-The-World）影响交互性能，V8 进一步引入了增量标记（Incremental Marking）、并发标记（Concurrent Marking）和并发清理（Concurrent Sweep），通过写屏障（Write Barrier）记录增量标记期间引用变更，保证正确性。

与前端工程师已有概念的对比：事件循环（Event Loop）是 JavaScript 的并发模型，负责调度任务，它决定了闭包和局部变量何时被释放；GC 是内存模型，决定堆中对象何时物理回收。二者在异步回调时交互——回调执行完毕，其栈帧弹出，对应引用消失，GC 才可能回收相关对象。而作用域链（Scope Chain）是静态的词法嵌套结构，编译期确定，它定义了变量的可见性；可达性则是动态的引用图，运行时由实际引用关系决定。前端常见的“闭包导致内存泄漏”实则是作用域链被外部引用链保持，GC 无法回收；这属于可达性而非作用域本身的规则。另外，WeakMap 的弱引用机制正是对可达性分析的补充：WeakMap 的键必须是对象，且不创建强引用，GC 在回收键对象时自动删除对应记录，这是引擎对 GC 语义的直接暴露。

### 3. 基础代码与实战验证
```text
// 以下代码验证弱引用与强引用对 GC 的影响，需在 Node.js 或浏览器控制台执行。

let obj = { payload: new Array(1000000) };

// 使用 WeakMap 持有引用
const wm = new WeakMap();
wm.set(obj, 'metadata');

// 使用 Map 持有引用
const map = new Map();
let obj2 = { payload: new Array(1000000) };
map.set(obj2, 'metadata');

// 释放外部强引用
obj = null;  // 此时 obj 不可达，WeakMap 中 key 的引用是弱引用，不阻止 GC。
             // 下一次 GC 后，对象及其 payload 会被回收，wm 中对应条目自动消失。

obj2 = null; // obj2 不可达，但 map 中 key 是强引用，对象仍然可达，不会被 GC 回收。
             // 必须显式执行 map.delete(obj2) 或 map.clear() 才能释放。

// 实际验证可手动触发 GC（Node.js 需 --expose-gc 启动）：
// global.gc();
// console.log(process.memoryUsage()); // 观察内存变化。

// 另一个示例：闭包中的引用链导致无法回收
function createLeak() {
  let bigData = new Array(1000000);
  return function leak() {
    // 内部函数引用 bigData，如果 leak 被外部长期持有，
    // 则 bigData 始终从根可达，无法被 GC。
    return bigData.length;
  };
}
const fn = createLeak();
// fn 持有闭包，闭包作用域持有 bigData，形成一条根->fn->闭包环境->bigData 的引用链。
```

### 4. 常见误区与进阶思考
误区 1：将变量赋值为 null 会立即触发垃圾回收。实际上，GC 的时机由引擎决定（如堆内存压力、空闲时间），赋 null 只是移除了一个引用，使对象从根变为不可达；真正的回收要等到下一次 GC 周期。尤其在现代分代 GC 中，老生代回收可能延迟很久，且增量标记阶段也可能存在短暂的引用残留。

误区 2：循环引用一定会导致内存泄漏。引用计数算法（如早期 IE 的 COM 实现）无法回收循环引用，但现代引擎都采用标记-清除，从根遍历可达性，循环引用中的对象如果没有外部引用，都不可达，会被正常回收。因此，只要没有根路径指向循环引用集合，GC 就能处理。

思考题：V8 分代 GC 的假设（弱代假说，Weak Generational Hypothesis）指出：大多数对象在年轻时就死亡。请解释这个假设如何影响新生代和老生代的回收频率、晋升策略，以及为什么增量标记需要写屏障来追踪老生代引用新生代的边？结合可达性分析的本质，说明为什么写屏障是必要的。
