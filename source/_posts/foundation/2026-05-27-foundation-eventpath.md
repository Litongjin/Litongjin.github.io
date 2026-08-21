---
title: "每日基础技术总结 · 2026-05-27 · 事件传播：捕获、冒泡、委托与 event.path"
date: 2026-05-27 08:00:00
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-05-27 · 事件传播：捕获、冒泡、委托与 event.path

## 📚 今日主题

> **事件传播：捕获、冒泡、委托与 event.path**（前端底层与计算机基础）

### 1. 核心概念速览
事件传播是浏览器DOM事件模型的核心机制，描述事件从触发点到DOM树根节点的完整流动路径。它由三个阶段构成：捕获阶段（capture phase）、目标阶段（target phase）、冒泡阶段（bubble phase）。捕获阶段从window/document沿DOM树向下传播至目标元素；目标阶段在目标元素上触发监听器；冒泡阶段从目标元素沿DOM树向上传播至window。事件委托则是利用冒泡阶段，在祖先节点统一监听子节点事件，从而减少监听器数量并动态适配新节点。event.path是Chrome等浏览器暴露的事件传播路径数组（非标准），标准API为Event.prototype.composedPath()。该机制解决了事件监听器的组织与调度问题，是浏览器事件循环、DOM操作、前端框架合成事件系统的基础。专业工程师必须掌握它，因为它是理解事件性能优化、跨浏览器兼容、框架事件实现（如React合成事件）以及调试复杂交互的根本前提。

### 2. 底层原理剖析
底层运行机制基于DOM树节点间的父子关系。当任意节点触发事件时，浏览器执行以下派发算法：
1. 从window对象开始，沿DOM树逐级向下，依次检查当前路径上的节点是否注册了该事件类型且`capture`为true的监听器，若有则按树深度顺序调用（捕获阶段）。
2. 到达事件目标节点，调用目标上所有监听器，不区分捕获或冒泡（目标阶段）。
3. 从目标的父节点开始，沿DOM树逐级向上，依次检查节点上是否注册了该事件类型且`capture`为false（即冒泡）的监听器，若有则按从叶到根的顺序调用（冒泡阶段）。

事件传播路径由事件发生瞬间的DOM树结构决定。浏览器内部维护一个`EventPath`列表，记录从window到目标节点的所有节点，以及shadow DOM中的相关节点。`event.path`就是这个列表的公开拷贝（非标准），而`composedPath()`返回该列表的标准版本，包含穿透shadow DOM的路径。

与前端已有概念的对比：原生DOM事件传播与React合成事件不同。React在根部（root）统一绑定事件，利用事件委托模拟捕获/冒泡，内部按组件树层级计算传播路径，而非依赖DOM树物理结构。React合成事件在冒泡阶段统一处理，并且其`stopPropagation`只能阻止合成事件在React组件树中传播，无法直接阻止原生事件传播（除非调用`nativeEvent.stopPropagation()`）。这与原生DOM事件传播存在本质差异，理解原生机制是理解框架事件系统的基石。

### 3. 基础代码与实战验证
```text
// 验证捕获与冒泡顺序以及 event.path 的生成
<!DOCTYPE html>
<html>
<head><meta charset="utf-8"><title>事件传播验证</title></head>
<body>
  <div id="outer" style="padding:20px;background:#eee">
    <button id="inner">点击</button>
  </div>
  <script>
    const outer = document.getElementById('outer');
    const inner = document.getElementById('inner');
    const log = msg => console.log(msg);

    // 捕获阶段：outer 注册捕获监听器（第三个参数 true）
    outer.addEventListener('click', function(e) {
      log('outer 捕获');
    }, true);

    // 冒泡阶段：outer 注册冒泡监听器（第三个参数 false 或省略）
    outer.addEventListener('click', function(e) {
      log('outer 冒泡');
    });

    // 目标阶段：inner 上同时注册捕获和冒泡监听器（目标阶段不区分捕获/冒泡，按注册顺序执行）
    inner.addEventListener('click', function(e) {
      log('inner 捕获注册（实际在目标阶段执行）');
    }, true);
    inner.addEventListener('click', function(e) {
      log('inner 冒泡注册（实际在目标阶段执行）');
    });

    // 点击 inner 后控制台输出顺序：outer 捕获 → inner 捕获注册 → inner 冒泡注册 → outer 冒泡
    // 证明捕获阶段先于目标阶段，目标阶段后是冒泡阶段。

    // 在冒泡阶段查看 event.path（Chrome）或 composedPath()（标准）
    document.addEventListener('click', function(e) {
      console.log('传播路径:', e.path || e.composedPath());
      // 输出 [inner, div#outer, body, html, document, Window]
    });
  </script>
</body>
</html>
```

### 4. 常见误区与进阶思考
误区1：认为`stopPropagation()`会阻止当前元素上所有监听器的执行。实际上，它只阻止事件继续传播到下一个阶段或下一个节点，当前节点上已注册的其他监听器仍会执行。若要彻底阻止当前节点上的剩余监听器，必须使用`stopImmediatePropagation()`。

误区2：将`event.path`当作标准API直接使用。`event.path`是Chrome私有实现，在Firefox等浏览器中不存在；标准方法是`event.composedPath()`，它返回相同含义的数组，且能正确处理shadow DOM。生产代码应优先使用`composedPath()`。

思考题：在目标元素上同时注册了捕获监听器和冒泡监听器，并且父元素的捕获监听器在捕获阶段调用了`stopPropagation()`，此时目标元素上的监听器还会执行吗？请基于事件传播三阶段的定义，分析`stopPropagation()`阻止的是传播路径的继续，还是目标阶段的触发？
