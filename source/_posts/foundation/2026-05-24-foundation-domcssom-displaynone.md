---
title: "每日基础技术总结 · 2026-05-24 · 渲染树构建：DOM/CSSOM 合并与 display:none"
date: 2026-05-24 08:00:00
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-05-24 · 渲染树构建：DOM/CSSOM 合并与 display:none

## 📚 今日主题

> **渲染树构建：DOM/CSSOM 合并与 display:none**（前端底层与计算机基础）

### 1. 核心概念速览
渲染树（Render Tree）是浏览器在 DOM 与 CSSOM 合并后生成的、仅包含可见节点的内部结构，用于驱动布局（Layout）与绘制（Paint）。其本质是 DOM 树中每个可见元素与 CSSOM 中匹配到的样式规则（Computed Style）的笛卡尔积的过滤投影。它解决的核心问题是：将两棵独立构建的树（DOM 描述结构、CSSOM 描述样式）合并为单一、可布局的树，同时剔除所有不会产生视觉输出的节点。机制上，浏览器从 DOM 根节点开始深度优先遍历，对每个节点在 CSSOM 中查找匹配的样式，若节点满足“不可见”条件（如 display:none、或属于 <head>、<meta> 等非渲染元素），则该节点及其子树整体从渲染树中剪除。该机制位于浏览器渲染管线的第二、三阶段之间（DOM/CSSOM 构建 → 渲染树构建 → 布局 → 绘制），是后续布局与绘制的前置依赖。专业工程师必须掌握它，因为性能优化（如避免强制同步布局）、CSS 选择器匹配策略、display 与 visibility 的语义差异、以及虚拟 DOM 的 diff 基础都根植于此。

### 2. 底层原理剖析
渲染树构建的输入是 DOM 树与 CSSOM 树。DOM 树是文档对象模型的层次化节点集合，CSSOM 是样式规则（Selector + Declaration）的树形结构。合并过程并非简单的属性复制，而是逐节点计算“最终生效样式”（Computed Style），即层叠、继承、默认值解析后的结果。关键算法逻辑如下：

1. 从 DOM 根节点（document）开始，按深度优先（先序）遍历所有节点。
2. 对每个节点：
   - 检查其是否为“非渲染”节点：
     * 节点类型为 DocumentType、Script、Link、Meta、Style、Title 等不产生视觉盒子的元素。
     * 节点自身或祖先节点的 computed style 中 display 值为 none。
     * 注意：visibility:hidden 或 opacity:0 不会从渲染树中移除，它们仍占据空间，只是不可见。
   - 若为非渲染节点，则跳过该节点及其所有后代（即剪枝）。
   - 若为渲染节点，则创建一个 RenderObject（渲染对象），关联该 DOM 节点及其 computed style。
3. 每个渲染对象还包含对其子渲染对象的引用，形成渲染树。渲染树中节点顺序与 DOM 树一致（除非存在绝对定位、浮动等需要特殊处理的脱离文档流的节点，这些在布局阶段处理）。
4. 当 DOM 或 CSSOM 发生变化时，浏览器会标记对应节点为 dirty，并增量重建渲染树（只影响受影响子树）。

用伪代码描述：

```
function buildRenderTree(domNode, inheritedDisplay):
  if domNode is not renderable: return null
  style = resolveStyle(domNode)  // 从 CSSOM 计算最终样式
  if style.display == 'none': return null
  renderObject = new RenderObject(domNode, style)
  for child in domNode.children:
    childRender = buildRenderTree(child, style.display)
    if childRender: renderObject.addChild(childRender)
  return renderObject
```

与前端已有概念的对比：这类似于“过滤器”模式——DOM 是“源数据”，CSSOM 是“映射规则”，渲染树是“过滤后的投影”。类比 Java 的接口与 TypeScript 的接口：Java 接口是运行时多态的行为契约（编译期类型约束，最终以字节码存在），而 TS 接口是纯编译期结构约束，运行时不保留。渲染树与 DOM 的关系类似：DOM 是运行时真实存在的树，渲染树是浏览器内部为布局而构建的临时结构，两者并非一一对应，且渲染树不包含非渲染节点，就像 TS 接口在编译后消失，但它在编译期影响了类型检查。本质区别在于：渲染树是“按需生成”的，每次样式变化都可能重建；而接口是静态的。

### 3. 基础代码与实战验证
以下用极简代码验证渲染树构建与 display:none 的剪枝行为。使用浏览器原生 API 观察布局副作用。

```html
<!DOCTYPE html>
<html>
<head>
  <style>
    .hidden { display: none; }
    .invisible { visibility: hidden; }
  </style>
</head>
<body>
  <div id="parent">
    <div id="child1" class="hidden">我不在渲染树中</div>
    <div id="child2" class="invisible">我在渲染树中，但不可见</div>
    <div id="child3">正常可见</div>
  </div>
  <script>
    // 验证 display:none 的节点不参与布局（无几何信息）
    const child1 = document.getElementById('child1');
    const child2 = document.getElementById('child2');
    const child3 = document.getElementById('child3');

    // offsetParent 为 null 表示该元素未生成渲染盒（不在渲染树中）
    console.log(child1.offsetParent); // null —— 渲染树剪枝
    console.log(child2.offsetParent); // body —— 仍在渲染树中

    // 强制获取布局信息，触发同步布局（reflow）
    const rect1 = child1.getBoundingClientRect();
    const rect2 = child2.getBoundingClientRect();
    console.log(rect1.width, rect1.height); // 0 0 —— 无几何尺寸
    console.log(rect2.width, rect2.height); // 实际宽高（如 0 0 取决于内容，但占据空间）

    // 验证 visibility:hidden 的节点仍占据布局空间
    const style2 = getComputedStyle(child2);
    console.log(style2.display); // 'block'
    console.log(style2.visibility); // 'hidden'

    // 注意：当 parent 被设为 display:none 时，所有子节点从渲染树移除
    document.getElementById('parent').style.display = 'none';
    console.log(child3.offsetParent); // null —— 祖先剪枝导致后代全部移除
  </script>
</body>
</html>
```

核心机制解释：`offsetParent` 返回最近的定位祖先，若元素不在渲染树中则返回 `null`。`getBoundingClientRect()` 在 `display:none` 下返回全零矩形，因为布局阶段根本未计算该元素。这直接验证了渲染树构建时对 `display:none` 的剪枝行为。

### 4. 常见误区与进阶思考
常见误区 1：认为 `display:none` 与 `visibility:hidden` 类似，都只是“看不见”。本质区别在于：`display:none` 导致元素及其子树从渲染树中整体移除，不生成任何盒模型，不参与布局，也不触发元素上的事件；`visibility:hidden` 仍生成渲染对象，参与布局，只是视觉上不可见，且其子元素可通过 `visibility:visible` 重新显示。这解释了为什么 `display:none` 的父元素无法通过 `getBoundingClientRect()` 获取尺寸，而 `visibility:hidden` 可以。

常见误区 2：认为渲染树构建是 DOM 树与 CSSOM 树合并时，每个 DOM 节点都对应一个渲染节点。实际上渲染树是过滤后的子集：非视觉元素（如 `<script>`）、`display:none` 的节点及其后代，以及 `::before`/`::after` 伪元素（生成额外渲染节点）都可能改变一一对应关系。另外，`position:absolute` 或 `float` 的元素在渲染树中仍存在，但布局阶段会脱离正常文档流，渲染树本身不处理这种差异。

思考题：假设一个 DOM 元素的 CSS 设置为 `display: contents`，它在渲染树中是否仍然存在？如果存在，其子元素如何渲染？请从渲染树构建的剪枝逻辑出发，分析 `display: contents` 与 `display: none` 的根本差异。提示：`display: contents` 使元素自身不生成盒子，但其子元素仍正常渲染——这需要在渲染树构建时如何处理该节点的“存在性”？
