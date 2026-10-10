---
title: "每日基础技术总结 · 2026-10-09 · DOM 事件传播模型：捕获阶段、目标阶段与冒泡阶段的触发时机与 stopImmediatePropagation"
date: 2026-10-09 08:00:00
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-09 · DOM 事件传播模型：捕获阶段、目标阶段与冒泡阶段的触发时机与 stopImmediatePropagation

## 📚 今日主题

> **DOM 事件传播模型：捕获阶段、目标阶段与冒泡阶段的触发时机与 stopImmediatePropagation**（前端底层与计算机基础）

### 1. 核心概念速览
DOM 事件传播模型是 W3C DOM Events 规范定义的树形事件派发机制：事件对象从 window 沿 target 的祖先链向下进入捕获阶段，到达 target 时进入目标阶段，再沿原路向上进入冒泡阶段。eventPhase 暴露当前阶段：1=CAPTURING_PHASE、2=AT_TARGET、3=BUBBLING_PHASE。目标阶段的关键规则是：target 上的监听器无论注册时 capture 是 true 还是 false，均按 addEventListener 的注册顺序被同步调用；capture 标志只在非目标节点的路径上决定监听器在哪个阶段触发。stopImmediatePropagation 是 EventTarget 事件监听器调用中的最强阻断原语：它同时阻止事件向后续节点传播，并阻止当前节点上尚未执行的剩余监听器；stopPropagation 仅阻止后续传播但保留同节点剩余监听器。该模型解决复杂 DOM 树中事件顺序、事件委托、合成事件分发、副作用隔离等问题，是浏览器事件系统与前端框架事件抽象的共同底层依据。专业工程师必须掌握它的原因在于：调试 handler 不触发、顺序异常、冒泡泄漏时，必须区分阶段、目标阶段注册顺序、stop 两种阻断的边界。

### 2. 底层原理剖析
事件派发前，浏览器根据 DOM 树构建 eventPath：从 window、document 一直到 target（含 target）。伪代码流程：
1. 捕获阶段：for node in eventPath[0..targetIndex-1] 按路径增序触发 capture=true 的监听器；若某监听器调用 stopPropagation 或 stopImmediatePropagation，则后续节点不再执行，后者还会停止当前节点剩余监听器。
2. 目标阶段：eventPhase=2。target 上的所有监听器按 addEventListener 注册先后同步执行，与 capture 参数无关。如果某监听器调用 stopPropagation，后续冒泡阶段取消，但当前 target 上后面的监听器仍会执行；如果调用 stopImmediatePropagation，则当前 target 上后面的监听器也被跳过。
3. 冒泡阶段：for node in eventPath[targetIndex-1..0]（从父节点到 window）触发 capture=false 的监听器，直到被阻断。
与 Node.js EventEmitter 的关键差异：EventEmitter 是单一 emitter 上的同步调用列表，没有 tree path、eventPhase、capture/bubble 阶段的区分，也没有 stopImmediatePropagation 等传播控制；浏览器 DOM 事件传播则是 EventTarget 树上的双向路径派发。与 React 17 合成事件对比：React 将监听器统一委托到 root 容器，按捕获/冒泡顺序遍历 fiber 树，并在合成事件对象上提供 stopPropagation/stopImmediatePropagation；其底层最终仍依赖原生 DOM 事件传播，但委托方式与原生混合时可能出现顺序差异。preventDefault 与传播完全正交，只标记 defaultPrevented，不影响事件继续在树中流动。

### 3. 基础代码与实战验证
```text
<div id='parent'><button id='child'>click</button></div>
<script>
  const order = [];
  function add(el, label, capture, immediateStop = false) {
    el.addEventListener('click', function handler(e) {
      if (immediateStop) e.stopImmediatePropagation(); // 当前 handler 会继续执行，但阻断同节点后续 listener 与后续传播
      order.push(`${label}:${capture ? 'CAPTURE' : 'BUBBLE'}=phase${e.eventPhase}`);
    }, capture);
  }

  const parent = document.getElementById('parent');
  const child = document.getElementById('child');

  add(parent, 'parent-capture', true);   // 捕获阶段在 target 之前触发，eventPhase=1
  add(child, 'child-capture', true);     // target 上第一个监听器；目标阶段按注册顺序执行，eventPhase=2
  add(child, 'child-bubble', false);     // target 上第二个监听器；虽然 capture=false，目标阶段仍在这里执行
  add(child, 'child-stop', false, true); // target 上第三个监听器；调用 stopImmediatePropagation
  add(child, 'child-after', false);      // target 上第四个监听器；应被跳过，验证同节点阻断
  add(parent, 'parent-bubble', false);   // 父级冒泡监听器；应被跳过，验证后续传播被阻断

  child.click();
  console.log(order);
  // 预期输出：
  // parent-capture:CAPTURE=phase1
  // child-capture:CAPTURE=phase2
  // child-bubble:BUBBLE=phase2
  // child-stop:BUBBLE=phase2
  // 不会输出 child-after 和 parent-bubble。
</script>
```

### 4. 常见误区与进阶思考
误区一：认为目标阶段遵循“先捕获后冒泡”。事实是：在 target 上，eventPhase 已经变为 2，捕获/冒泡标志不再参与排序；所有 target 监听器严格按 addEventListener 注册顺序同步执行。若在 target 先注册 capture=false 监听器，再注册 capture=true 监听器，则前者先执行。
误区二：混淆 stopPropagation 与 stopImmediatePropagation。stopPropagation 仅停止向后续节点传播，当前节点上后续监听器仍会执行；只有 stopImmediatePropagation 会同时阻止当前节点后续监听器。另一个常见误用是认为 preventDefault 会阻止传播，其实它只取消默认行为，事件仍继续传播。
进阶思考：target 上的一个捕获监听器调用 stopPropagation 后，target 上后面注册的冒泡监听器还会执行吗？父级冒泡监听器呢？请结合 eventPhase 与注册顺序解释。若你能说明“target 后续监听器会执行，因为它们仍处于同一目标阶段；父级冒泡不会执行，因为传播在目标阶段已被 stopPropagation 终止”，则说明已理解两种 stop 的边界。
