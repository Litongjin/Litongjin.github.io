---
title: "每日基础技术总结 · 2026-09-16 · 内存泄漏排查"
date: 2026-09-16 07:02:04
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-16 · 内存泄漏排查

## 📚 今日主题

> **内存泄漏排查**（前端底层与计算机基础）

### 1. 核心概念速览
### 1. 核心概念速览
内存泄漏（memory leak）在 GC 托管语言中的严谨定义：从 GC Roots 出发，经强引用边仍可达，但对象在程序语义上已不会再被使用；可达性分析不区分“仍可用”与“未来可能用”，因此该对象占据的内存无法回收。本质是对象图可达性错误与生命周期不匹配，而不是“忘记 free”。
解决的核心问题：定位并切断从根到无用对象的强引用路径，或调整持有者生命周期，使对象在业务生命周期结束后进入不可达状态。机制：V8 等引擎周期性执行可达性标记；根集合包括 globalThis、活动栈帧、闭包环境、微任务/宏任务队列、定时器注册表、事件监听注册表、未完成 I/O/原生句柄等。弱引用边（WeakMap/WeakSet/WeakRef）不参与根可达性判定。
体系位置：位于语言引用语义、运行时 GC、宿主环境资源管理、性能与稳定性工程的交叉层。浏览器侧关联 SPA 长生命周期、DOM detached、事件监听、WebSocket、闭包缓存；Node 侧关联常驻进程、连接池、流、Buffer/ArrayBuffer 外部内存；AI 侧关联数据管道、推理服务常驻内存、Python 引用计数+GC 与 GPU/主机内存容量。专业工程师必须掌握，因为泄漏表现为堆增长、GC 停顿增加、吞吐下降、OOM、成本上升，且仅靠重启无法根治。

### 2. 底层原理剖析
### 2. 底层原理剖析
GC 可达性模型：
roots = { globalThis, active_stack_frames, closure_envs, microtask_queue, timer_registry, event_listener_registry, pending_io_handles, native_handles }
mark:
  worklist = roots
  while worklist:
    o = pop(worklist)
    if o.marked == false:
      o.marked = true
      for r in o.strong_refs:
        if r.marked == false:
          push(worklist, r)
sweep:
  for o in heap:
    if o.marked == false:
      free(o)
WeakMap/WeakSet/WeakRef 边不加入 strong_refs；因此弱引用目标即使从根不可达，也可被回收。
V8 三色标记与写屏障：white 未发现，grey 已发现但子引用未扫描，black 已发现且子引用已扫描。增量/并发标记时，mutator 可能修改对象图。若 black 对象新增指向 white 对象的强引用而不通知 GC，white 对象会被漏标并错误回收。写屏障修正：
write_barrier(parent, field, child):
  if parent.color == BLACK and child.color == WHITE:
    child.color = GREY
    push(worklist, child)
这保证新引用被重新扫描。
分代与回收：新生代 Scavenge 使用 from/to 半空间复制，存活对象复制到 to，经历两次仍存活晋升老生代；老生代 Mark-Sweep-Compact，增量/并发标记，写屏障维护三色不变式。WeakMap 是 ephemeron：key 不可达时 entry 的 value 不再作为根可达对象；WeakRef.deref() 可能在任意 GC 后返回 undefined；FinalizationRegistry 回调时机不确定，仅用于清理通知，不用于业务正确性。
排查机制：堆快照对比。基线快照 -> 复现操作 N 次 -> 强制 GC -> 第二快照 -> Comparison 视图看新增对象与 retained size -> 对增长对象看 Retainers，找到从 GC Roots 到对象的强引用路径 -> 切断路径或改为弱引用/缩短生命周期。关键指标：Shallow Size 是对象自身大小；Retained Size 是回收该对象后可释放的总大小；Dominator Tree 表示支配关系；Distance 是到根最短强引用距离。Node 侧：process.memoryUsage()、v8.getHeapStatistics()、--expose-gc + global.gc()、--trace-gc、--inspect、heapdump、clinic。
对比前端已有概念：事件循环中的任务队列、定时器注册表、事件监听注册表都是 GC Roots 的一部分。回调闭包被注册表强引用时，闭包环境不会因函数执行结束而释放。TS 接口是编译期结构类型，擦除后不产生运行时对象；Java 接口是运行时类型契约，有方法表与类元数据；二者说明类型系统不直接决定对象图可达性。内存泄漏取决于运行时强引用边，而不是类型标注。C/C++ 的 malloc/free 是显式生命周期；JS/Java 是可达性生命周期。前者的泄漏是缺少 free，后者的泄漏是多余强引用。React useEffect cleanup、AbortController、removeEventListener、clearInterval 本质都是切断注册表到回调闭包的强引用路径。

### 3. 基础代码与实战验证
```text
### 3. 基础代码与实战验证
Node 验证：保存为 leak.js，执行 node --expose-gc leak.js。
const leaks = [];
function retainPayload() {
  const payload = new Array(1_000_000).fill(0);
  leaks.push(payload); // leaks 是模块级绑定，位于 GC Roots 可达链；payload 被强引用，mark 阶段必然可达
}
for (let i = 0; i < 8; i++) {
  retainPayload();
}
console.log('after alloc', process.memoryUsage().heapUsed);
if (global.gc) global.gc(); // 强制全量 GC；无 --expose-gc 时为 undefined
console.log('after gc', process.memoryUsage().heapUsed); // heapUsed 不显著下降，因为 leaks 仍强引用 payload

弱引用对照：
const weakRegistry = new WeakMap();
function attach(obj) {
  weakRegistry.set(obj, new Array(1_000_000).fill(0)); // key 为弱引用；不阻止 obj 被回收
}
let target = {};
attach(target);
target = null; // 断开从根到 target 的强引用；WeakMap 的 key 不再强可达
if (global.gc) global.gc();
console.log('after weak gc', process.memoryUsage().heapUsed); // entry 后续可被清理；FinalizationRegistry 回调时机不确定

Chrome DevTools 验证步骤：
1. Memory 面板先拍 Heap snapshot 作为基线。
2. 执行可疑交互 N 次，例如打开/关闭弹窗、路由切换、订阅/取消订阅。
3. 点击垃圾桶图标强制 GC，再拍第二张 Heap snapshot。
4. 选择 Comparison，按 Size Delta / #New 排序，定位持续新增且 retained size 大的对象。
5. 选中对象查看 Retainers，路径通常为 globalThis -> 变量名 / EventTarget -> listener -> closure / Timer -> callback / DOM node -> detached tree。
6. 在代码中切断对应强引用，重复相同快照流程验证 retained size 不再随操作次数线性增长。
注意：heapUsed 只反映 V8 堆，不包含 Buffer/ArrayBuffer 等外部内存；需要同时看 external、arrayBuffers、rss。
```

### 4. 常见误区与进阶思考
### 4. 常见误区与进阶思考
误区 1：强制 GC 后 heapUsed 没降就断言泄漏。强制 GC 只回收不可达对象；若对象仍从 GC Roots 强可达，heapUsed 不会降。内存未降还可能来自缓存预热、JIT 代码、堆碎片、外部内存、GC 延迟。正确方法是比较多次操作后的 retained size 趋势与 retainer path。
误区 2：把 WeakMap/WeakRef 当成通用释放机制。WeakMap 弱化的是 key 到 entry 的可达性，value 仍是强引用；若 value 被其他根强引用，或业务仍持有 WeakRef 目标的其他强引用，对象仍不可回收。FinalizationRegistry 回调时机不确定，不能依赖它释放关键资源。
进阶思考题：在 Chrome DevTools 的 Retainers 面板中，如何仅凭从 GC Roots 到泄漏对象的路径，区分以下三类泄漏并给出修复策略：全局变量/模块级缓存、未移除的事件监听器、已从 DOM 树移除但被 JS 引用的 detached DOM 节点？要求指出路径中关键节点（globalThis、EventTarget、listener、closure、DOM node、detached tree）以及应切断哪条强引用边。
