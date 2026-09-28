---
title: "每日基础技术总结 · 2026-09-29 · Web 字体加载的 FOIT/FOUT 与字体交换机制"
date: 2026-09-29 07:20:08
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-29 · Web 字体加载的 FOIT/FOUT 与字体交换机制

## 📚 今日主题

> **Web 字体加载的 FOIT/FOUT 与字体交换机制**（前端底层与计算机基础）

### 1. 核心概念速览
FOIT（Flash of Invisible Text，不可见文本闪烁）与 FOUT（Flash of Unstyled Text，未样式化文本闪烁）是浏览器在 Web 字体网络加载完成前渲染文本的两种竞态现象。其底层原因是：文本绘制必须依赖字体的度量数据（字形宽度、基线、字距）完成 layout 与 paint，但字体文件本身是异步网络资源；当一帧需要被绘制时字体尚未就绪，浏览器必须二选一——隐藏文本，或先用回退字体绘制。隐藏文本就是 FOIT，用回退字体后等待自定义字体到达再替换就是 FOUT。该策略由 CSS Fonts Level 4 的 font-display 属性显式控制，本质是一个基于字体加载状态机（unloaded/loading/loaded/failed）和两个时间窗（blockPeriod/swapPeriod）的渲染分派器。它解决的核心问题是：在不确定的字体加载延迟与必须输出的每一帧之间，决定何时暴露文本、何时切换字体。该机制位于浏览器网络栈与 Layout/Paint 管线的交界处，直接决定 LCP 与 CLS 两大核心性能指标。专业工程师必须掌握它，因为 preload、字体子集化、度量覆盖等一切优化手段，本质上都只是在调整这个状态机的下载时机、时间窗和切换代价，而不是绕过它。

### 2. 底层原理剖析
浏览器解析到 @font-face 时，会创建 FontFace 对象并加入 FontFaceSet，但不会阻塞 DOM 解析。只有当某个元素计算样式后的 font-family 链中真正用到该字体时，Layout 阶段才会向字体对象索取度量数据；若字体处于 loading，浏览器必须立即做出当前帧的绘制决策。

font-display 各值等价于不同分派策略：

- block：loading 期间在 blockPeriod 内绘制透明文本（FOIT）；超时后用回退字体；字体加载完成后仍会替换并重排。
- swap：blockPeriod 为 0，直接绘制回退字体（FOUT）；字体加载完成后无条件替换并重排。
- fallback：短 blockPeriod（约 100ms）内隐藏，随后用回退字体；但一旦超过 swapPeriod（约 3s），字体即使加载完成也不再替换。
- optional：几乎不隐藏也不交换；若字体不在本地缓存或网络条件不佳，浏览器可完全跳过下载。
- auto：等价于 block，但超时由 UA 决定。

逐帧决策的伪代码如下：

if state == loaded:
    drawWithWebFont()
else:
    if fontDisplay in (block, fallback) and elapsed < blockPeriod:
        drawTransparent()          # FOIT
    else:
        drawWithFallback()         # FOUT
        # 字体下载完成后触发 FontFaceSet 事件
        if fontDisplay in (block, swap):
            invalidateLayoutAndRedraw()
        elif fontDisplay == fallback and elapsed < swapPeriod:
            invalidateLayoutAndRedraw()
        else:
            keepFallback()         # 永久放弃交换

关键差异在于：字体文件下载完成会使相关文本的 Layout 失效，因此 swap 和 block 在字体替换时都会产生一次二次布局偏移（CLS）。与前端已有模型对比：它不是 `<img>` 那种有固有占位尺寸的替换元素，也不像 JS 的 async/defer 只调整执行顺序而不改变视觉状态；它更接近 React Suspense 的 fallback——但 CSS 没有协调器，必须在当前帧立刻输出真实绘制内容，所以只能在隐藏与回退字体之间二选一。

### 3. 基础代码与实战验证
```text
以下是最小可验证页面。配合 DevTools Network 面板的 Slow 3G 和禁用缓存，对比将 font-display 改为 swap 与 block 时首帧文本是否可见。

<!DOCTYPE html>
<html lang='zh-CN'>
<head>
  <meta charset='utf-8'>
  <title>FOIT / FOUT 验证</title>
  <style>
    @font-face {
      font-family: 'Slow';
      src: url('/slow.woff2') format('woff2');
      /* 关键值：swap → blockPeriod 为 0，首帧立即使用回退字体绘制 */
      font-display: swap;
    }
    body {
      font-family: 'Slow', serif;
      font-size: 3rem;
    }
  </style>
</head>
<body>
  <h1 id='probe'>AAAA</h1>
  <script>
    // 记录首帧布局宽度：此时字体处于 loading，必然使用回退字体的度量
    const probe = document.getElementById('probe');
    const fallbackWidth = probe.getBoundingClientRect().width;
    console.log('fallback 渲染宽度：', fallbackWidth);

    // document.fonts.ready 在所有字体加载完成后 resolve
    // 若 font-display: swap，此刻字体已替换并触发 re-layout，宽度会改变 → 证明 FOUT 已发生
    document.fonts.ready.then(() => {
      const webWidth = probe.getBoundingClientRect().width;
      console.log('Web 字体渲染宽度：', webWidth);
      console.log('宽度变化：', webWidth - fallbackWidth);
    });
  </script>
</body>
</html>
```

### 4. 常见误区与进阶思考
误区 1：只把 `font-display: swap` 当成加载体验的银弹，忽略字体替换时的度量突变。swap 只是将 FOIT 转化为 FOUT，在字体到达时同样触发一次 Layout 失效；如果自定义字体与回退字体的字宽、行高差异大，CLS 反而比 FOIT 更严重。真正的优化必须配合 `size-adjust`、`ascent-override` 等字体度量覆盖，或使用 preload 缩小交换窗口。

误区 2：混淆 `fallback` 与 `optional` 的交换策略。`fallback` 只在首次加载的极短 swapPeriod 内完成下载才会交换；一旦超过窗口，即使字体随后下载完成也永久放弃交换，避免页面在已经很稳定的布局上再次跳动。`optional` 更加激进：浏览器在必要时可以完全跳过下载，页面彻底使用回退字体，很多工程师误以为 optional 是“尽力用字体”，实际它是“尽力不用字体”。

思考题：一个高延迟用户访问你的站点，字体在 4 秒时下载完成，但你配置的是 `font-display: fallback`，最终页面一直保持回退字体。请从字体状态机与 swapPeriod 的关系解释为什么，并说明若要保证这类用户最终能使用 Web 字体，应当调整哪个环节——是改用 `swap + size-adjust`，还是保持 `fallback + preload`？
