---
title: "每日基础技术总结 · 2026-11-07 · CSS Logical Properties：书写模式（Writing Mode）对方向性布局的支持"
date: 2026-11-07 07:01:31
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-07 · CSS Logical Properties：书写模式（Writing Mode）对方向性布局的支持

## 📚 今日主题

> **CSS Logical Properties：书写模式（Writing Mode）对方向性布局的支持**（前端底层与计算机基础）

### 1. 核心概念速览
CSS Logical Properties 是一组以流相对方向（flow-relative）而非物理方向（left/right/top/bottom）表达盒模型、尺寸、定位、边框、内边距、外边距、文本对齐等属性的 CSS 模块。它把样式绑定到当前 writing-mode 与 direction 定义的 inline axis 和 block axis：inline-start/inline-end 表示行内轴起止边，block-start/block-end 表示块轴起止边。writing-mode 决定块轴方向与行内轴方向（horizontal-tb、vertical-rl、vertical-lr 等），direction 决定行内基方向（ltr/rtl）。它解决的问题是：同一组件在 LTR/RTL、横排/竖排（尤其 CJK 竖排）下无需重写 left/right/top/bottom 即可保持方向正确的布局。机制是样式解析后由浏览器根据元素的计算 writing-mode/direction 将逻辑属性映射到物理边，再由布局引擎按物理几何计算。它位于浏览器渲染管线的样式计算与布局层，是人机界面国际化、双向文本与书写模式适配的基础设施；不属于 AI 算法层，但设计系统、组件库与跨语言 Web 产品若硬编码物理方向，会在 RTL、竖排和混合书写模式下产生不可维护的分支，因此专业工程师必须掌握。

### 2. 底层原理剖析
核心模型：
1. 两个轴：inline axis 为文本行内推进方向；block axis 为块级盒子堆叠方向。
2. 两个方向：start/end 由 writing-mode + direction 共同决定。
3. 逻辑属性：margin/padding/border/inset/inline-size/block-size/text-align:start|end 等。

映射伪代码：
map(logical, el):
  wm = computed(writing-mode)
  dir = computed(direction)
  if wm == horizontal-tb:
    inlineAxis = horizontal
    blockAxis = vertical
    blockStart = top; blockEnd = bottom
    inlineStart = (dir == ltr) ? left : right
    inlineEnd = opposite(inlineStart)
  if wm == vertical-rl:
    inlineAxis = vertical
    blockAxis = horizontal
    blockStart = right; blockEnd = left
    inlineStart = (dir == ltr) ? top : bottom
    inlineEnd = opposite(inlineStart)
  if wm == vertical-lr:
    blockStart = left; blockEnd = right
    inlineStart = (dir == ltr) ? top : bottom
  resolve:
    margin-inline-start -> margin-{inlineStart}
    padding-block-start -> padding-{blockStart}
    inline-size -> size along inlineAxis
    block-size -> size along blockAxis
    inset-inline-start -> offset from inlineStart edge

解析流程：声明 -> 计算 writing-mode/direction -> 逻辑属性映射 -> 物理几何 -> 布局。writing-mode 改变的是文档流的块轴与行内轴，影响行盒生成、字形方向、滚动方向和布局，不是 transform 的视觉旋转。writing-mode 与 direction 可继承，映射随计算值沿树传播。

与已有概念对比：
- 与 flexbox 的 main-axis/cross-axis 同构：flex 用 main/cross 避免 left/right，logical properties 用 inline/block 避免物理边；区别是 flex 轴由容器的 flex-direction 定义，logical 轴由元素的 writing-mode/direction 定义，且前者是单一布局算法内部概念，后者是属性级方向映射。
- 与 em/rem 类似：都是把值绑定到上下文而非固定物理量；但 logical properties 绑定的是书写模式上下文。
- 与 transform: rotate 不同：transform 只改变绘制，不改变布局流与逻辑属性映射。
- 与 CSS 物理属性相比：margin-left 是绝对物理边，margin-inline-start 是流相对边；同一逻辑声明在不同 direction/writing-mode 下可映射到不同物理属性。

### 3. 基础代码与实战验证
```text
<!doctype html>
<meta charset='utf-8'>
<style>
  .demo {
    /* 逻辑属性不写 left/right/top/bottom，而是写流相对边 */
    writing-mode: horizontal-tb;
    direction: ltr;
    inline-size: 220px;        /* horizontal-tb 下映射为 width */
    block-size: 72px;          /* horizontal-tb 下映射为 height */
    margin-inline-start: 24px; /* ltr 下映射为 margin-left；rtl 下变为 margin-right */
    padding-block-start: 12px; /* horizontal-tb 下映射为 padding-top */
    border-inline-start: 4px solid #333; /* ltr 下映射为 border-left */
    background: #eee;
  }
  .rtl {
    direction: rtl; /* 仅改变行内基方向，inline-start 映射到右侧 */
  }
  .vertical {
    writing-mode: vertical-rl; /* 块轴变为从右到左，行内轴为从上到下 */
    direction: ltr;
    /* 此时 inline-size 映射为 height，block-size 映射为 width；
       margin-inline-start 映射为 margin-top；
       padding-block-start 映射为 padding-right；
       border-inline-start 映射为 border-top。 */
  }
</style>
<div class='demo'>horizontal-tb / ltr</div>
<div class='demo rtl'>horizontal-tb / rtl</div>
<div class='demo vertical'>vertical-rl / ltr</div>

验证方式：在 DevTools 的 Computed 面板查看每个元素的 width/height、margin-left/right、padding-top/right、border-left/top；切换 direction 或 writing-mode 时，CSS 声明不变，但物理映射改变。
```

### 4. 常见误区与进阶思考
误区一：把 inline/block 当成 display:inline/block。inline-size 不是 inline 元素的尺寸，而是书写模式中行内轴尺寸；horizontal-tb 下等于 width，vertical-* 下等于 height。误区二：以为 logical properties 只服务 RTL。它同时服务竖排 CJK、sideways、混合书写模式和未来书写模式；且 transform: rotate 不会改变逻辑映射，因为 transform 不改变布局流。另有实现细节：只设 CSS direction 而忽略 HTML dir、unicode-bidi 与 bidi 隔离，复杂双向文本仍会出错。

思考题：在 writing-mode: vertical-rl; direction: rtl; 的元素上，inset-inline-start: 10px 与 inline-size: 100px 分别落到哪条物理边、哪个物理尺寸？再叠加 transform: rotate(90deg) 后答案是否改变？为什么？
