---
title: "每日基础技术总结 · 2026-09-27 · CSS 层叠上下文与 z-index 堆叠规则"
date: 2026-09-27 07:03:48
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础（扩展）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-27 · CSS 层叠上下文与 z-index 堆叠规则

## 📚 今日主题

> **CSS 层叠上下文与 z-index 堆叠规则**（前端底层与计算机基础（扩展））

### 1. 核心概念速览
CSS 层叠上下文（stacking context）是浏览器渲染引擎在合成阶段为元素分配的抽象层叠容器，它定义了元素及其后代在 z 轴上的排序范围。本质是一个原子化隔离环境：在同一层叠上下文内部，元素按层叠级别排序；外部任何 z-index 数值不能越过该上下文与内部后代比较。它解决的问题是：当页面元素在空间上重叠时，浏览器如何确定自底向上的绘制顺序。机制是通过特定属性（positioned + z-index 非 auto、opacity 非 1、transform 非 none、filter、will-change、contain 等）触发该上下文，将其中所有元素归入一个局部排序组，组与组之间按组所在上下文的层叠级别参与全局排序。该机制位于浏览器渲染管线的『合成（compositing）』阶段，与图层提升、GPU 纹理合成直接关联。专业工程师必须掌握它，因为视觉层叠错误常不直接体现在 DOM 结构里，且与图层爆炸等性能问题强相关；理解它才能在复杂 UI 中预测覆盖关系，并避免因无谓的上下文创建导致额外合成开销。

### 2. 底层原理剖析
底层机制可抽象为两步递归排序：

第一步，上下文识别：遍历 DOM 元素，若满足任一条件则创建层叠上下文：
- position 为 relative/absolute/fixed/sticky 且 z-index 非 auto；
- opacity 小于 1；
- transform、perspective、filter、backdrop-filter、mix-blend-mode、mask 等非初始值；
- will-change 指定上述任一属性；
- 根元素 html 本身形成根层叠上下文。

第二步，在同一父上下文内，按固定优先级自底向上绘制：
0. 父元素的背景和边框；
1. 负 z-index 的层叠上下文（按 z-index 数值升序，同值按 DOM 顺序）；
2. 块级盒子的非浮动后代；
3. 浮动盒子和内联内容；
4. z-index 为 auto 或 0 的定位元素；
5. 正 z-index 的层叠上下文（按 z-index 数值升序，同值按 DOM 顺序）。

伪代码：
function paintStackTree(ctx) {
  sort children by layer category;
  paint(ctx.backgroundAndBorders);
  paint(negativeZOrderedChildren);   // 内部递归
  paint(nonFloatedBlockChildren);
  paint(floatsAndInlineChildren);
  paint(autoPositionedChildren);
  paint(positiveZOrderedChildren);   // 内部递归
}

递归的关键是：每个拥有层叠上下文的子节点，在父级看来是一个原子层；其内部排序再递归进行。

与前端已有概念的对比：z-index 不是全局坐标，而是相对于最近的祖先层叠上下文，这与 position 的 containing block 类似；同时，层叠上下文与 BFC 类似，都是对某一维度（z 轴）的包含隔离，但 BFC 约束水平/垂直布局，层叠上下文约束 z 轴排序。更深层地理解，层叠上下文是渲染作用域，类似 JS 的词法作用域，只不过它不产生继承而只产生隔离，这正是 identity 与 isolation 的本质差别。

### 3. 基础代码与实战验证
```text
<style>
  .parent { position: relative; z-index: 0; } /* positioned + 非 auto 值创建一个层叠上下文 */
  .inner { position: absolute; z-index: 9999; } /* 此值只绑定在 .parent 的内部排序中 */
  .overlay { position: fixed; z-index: 1; } /* 与 .parent 整体比较 */
  .stack { position: relative; z-index: 0; }
  .stack > div { position: absolute; width: 100px; height: 100px; }
  .stack .a { z-index: 2; background: #f00; }
  .stack .b { z-index: 1; background: #00f; }
</style>

<!-- 实验一：验证层叠上下文的隔离性 -->
<div class='parent' style='background:#eee; width:200px; height:100px;'>
  <div class='inner' style='background:#f00;'>inner</div>
</div>
<div class='overlay' style='background:#00f; width:100px; height:100px;'>overlay</div>

关键注释：overlay 的 z-index 为 1，parent 的 z-index 为 0，因此整体上 overlay 覆盖 parent；inner 的 z-index 9999 不能越过 parent 的边界，因为 inner 被限制在 parent 创建的层叠上下文内。直观结果：你看到蓝色 overlay 在红色 inner 之上。

<!-- 实验二：验证同一上下文内的正规排序规则 -->
<div class='stack' style='width:200px; height:100px; margin-top:10px;'>
  <div class='a'>a(2)</div>
  <div class='b'>b(1)</div>
</div>

关键注释：.stack 自身创建层叠上下文，a 和 b 都是其定位后代。根据优先级规则，正 z-index 子上下文按数值升序绘制，因此 b 先绘制，a 后绘制，最终 a 叠在 b 之上。两者处于同一上下文，比较才有意义。
```

### 4. 常见误区与进阶思考
误区 1：把 z-index 当作全局优先级。
实际 z-index 只在同一个层叠上下文内生效。若某个父级上下文在整体层级中低于另一个外部元素，那么父级内部任意大的 z-index 都无法翻盘。常见场景：父元素设了 opacity: 0.99 或 transform，无意间创建了上下文，子元素弹层的 z-index 成了摆设，被外部遮罩覆盖。

误区 2：认为只有 position + z-index 才能创建层叠上下文。
opacity<1、transform、filter、backdrop-filter、mix-blend-mode、will-change、contain 等都会创建。尤其是 opacity 和 transform 常用在动画中，一个元素只是在做透明度渐入，其子元素的层叠关系因此被隔离，导致复杂的堆叠异常。

思考题：在同一个层叠上下文内，有一个定位元素 A（position: absolute; z-index: 0; background: #333），其内部包含一个后代 B（position: absolute; z-index: -1; background: #fff）。请推导 A 的背景与 B 的绘制顺序。若存在一个同级元素 C（z-index: auto）与 A 重叠，C 的文本内容绘制在 A 的背景之上还是之下？请用上文的优先级列表解释。
