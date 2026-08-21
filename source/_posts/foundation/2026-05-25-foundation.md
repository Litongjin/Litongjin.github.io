---
title: "每日基础技术总结 · 2026-05-25 · 层叠上下文与绘制顺序"
date: 2026-05-25 08:00:00
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-05-25 · 层叠上下文与绘制顺序

## 📚 今日主题

> **层叠上下文与绘制顺序**（前端底层与计算机基础）

### 1. 核心概念速览
层叠上下文（Stacking Context）是 CSS 渲染引擎在绘制阶段用于决定元素在 Z 轴（垂直于视口）上合成顺序的局部环境。它由特定属性触发，例如定位元素（position 非 static）且 z-index 不为 auto、opacity 小于 1、transform 不为 none、filter 不为 none 等。其本质是对元素进行分组：每个层叠上下文内部维护一套完整的层叠顺序，上下文作为一个整体参与外层上下文的排序。解决的问题是：当多个元素在空间上重叠时，确定覆盖关系的唯一依据。机制上，浏览器绘制时按深度优先递归遍历层叠上下文树，对每个节点执行固定的绘制顺序（背景、负 z-index、块级、浮动、行内、定位、正 z-index）。在浏览器整体体系中，它位于渲染流水线的绘制与合成阶段，与图形学中的深度排序、遮挡剔除同构。专业工程师必须掌握它，因为它是控制视觉层级最底层的手段，任何动画、弹层、复杂 UI 的覆盖关系都受其约束，不理解它就无法预测和调试渲染结果。

### 2. 底层原理剖析
层叠上下文的创建条件（任一即可）：
- 根元素（html）
- position 非 static 且 z-index 非 auto
- opacity 小于 1
- transform 非 none
- filter 非 none
- will-change 指定上述属性
- 其他如 mix-blend-mode、isolation 等

绘制顺序（针对单个层叠上下文内部）：
1. 该元素的背景和边框
2. 负 z-index 的后代层叠上下文
3. 块级盒子（非定位、非浮动）
4. 浮动盒子
5. 行内盒子
6. z-index:auto 或 0 的定位元素（其中 z-index:0 会创建新的层叠上下文）
7. 正 z-index 的后代层叠上下文

伪代码（文字描述）：
paint(stackingContext):
  paint background/border of context root
  for each child in negative z-index order: paint(child)
  for each block-level child: paint(child)
  for each float child: paint(child)
  for each inline child: paint(child)
  for each positioned child with z-index auto/0: paint(child)
  for each child in positive z-index order: paint(child)

注意：同一个层叠上下文内，兄弟节点的 z-index 决定绘制先后；不同层叠上下文之间，父上下文的层叠级别决定整体先后，子节点的 z-index 无法跨父级比较。

与前端已有概念的对比：
- 与 BFC（块格式化上下文）的对比：BFC 是布局阶段的二维环境，约束盒子在水平/垂直方向的排列和浮动隔离；层叠上下文是绘制阶段的三维环境，约束元素在 Z 轴的合成顺序。两者都是渲染引擎的局部抽象，但作用域不同，且不互相替代。
- 与 Java 接口和 TS 接口的差异类似，不同概念看似相近（都叫上下文），但本质定义和运行时行为完全不同。BFC 影响布局尺寸和位置，层叠上下文影响绘制先后和遮挡，必须从渲染阶段区分。

### 3. 基础代码与实战验证
```text
以下代码验证：子元素的 z-index 无法超过父层叠上下文的整体层级。

<!DOCTYPE html>
<style>
  .parent {
    position: relative;
    z-index: 1; /* 创建层叠上下文，整体层级为 1 */
  }
  .child {
    position: absolute;
    z-index: 999; /* 子元素 z-index 极高，但受限于父上下文 */
  }
  .other {
    position: absolute;
    z-index: 2; /* 与 .parent 同层，层级为 2 */
  }
</style>

<div class='parent'>
  <div class='child'>z-index 999 的子元素</div>
</div>
<div class='other'>z-index 2 的兄弟元素</div>

预期结果：虽然 .child 的 z-index 为 999，但 .parent 的整体层叠级别为 1，小于 .other 的 2，因此 .other 覆盖 .child。这证明 z-index 只在同一层叠上下文内有效，跨上下文时以父上下文的层级为准。
```

### 4. 常见误区与进阶思考
常见误区 1：认为 z-index 是全局的，数值越大一定显示在越上层。实际上 z-index 的作用范围仅限于同一个层叠上下文。当元素位于不同的层叠上下文中时，比较的是各自所属上下文的层叠级别，子元素的 z-index 无法参与跨上下文比较。

常见误区 2：认为 z-index: auto 与 z-index: 0 完全相同。区别在于：z-index: 0 会创建一个新的层叠上下文，而 z-index: auto 不会。这导致一个关键差异：一个定位元素如果 z-index: 0，其内部负 z-index 的子元素会被限制在该元素创建的上下文中，绘制在该元素背景之后；而如果 z-index: auto，该元素不创建上下文，内部负 z-index 子元素可能跑到祖先元素的背景之前。

进阶思考题：有两个兄弟 div，第一个 div 设置 position: relative; z-index: 0，第二个 div 设置 position: relative; z-index: auto。它们的子元素中有一个负 z-index 的绝对定位元素。请分析：这两个负 z-index 子元素分别会绘制在哪个层级？为什么？这个问题的答案直接揭示层叠上下文与 z-index 的底层交互机制。
