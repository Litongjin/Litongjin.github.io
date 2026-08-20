---
title: "每日基础技术总结 · 2026-06-12 · DOM-based XSS 与 innerHTML 的突变 XSS (mXSS)"
date: 2026-06-12 08:00:00
categories: [技术分享]
tags: ["技术分享", "安全基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-06-12 · DOM-based XSS 与 innerHTML 的突变 XSS (mXSS)

## 📚 今日主题

> **DOM-based XSS 与 innerHTML 的突变 XSS (mXSS)**（安全基础）

### 1. 核心概念速览
DOM-based XSS 是一类完全在浏览器 DOM 环境中发生的注入攻击，其本质是：不可信数据经由 JavaScript 的 DOM 操作 API（如 innerHTML、document.write、outerHTML、insertAdjacentHTML 等）被解析为 HTML，导致攻击者控制的标记在文档上下文中被执行。它不同于反射型/存储型 XSS 的服务器端注入，漏洞根源在于客户端代码将数据拼接进 HTML 解析器。mXSS（mutation XSS）是 DOM-based XSS 的进阶变体，其机制是：浏览器在解析、序列化、再解析 HTML 的过程中，对某些畸形或边界标记（如 <noscript>、<svg>、<math>、<!-- 注释）进行语义修正（mutation），使得原本被过滤或转义后看似安全的字符串，在第二次被设置到 innerHTML 时发生语义变化，重新激活为可执行代码。它解决的核心问题是：单纯依赖过滤黑名单、转义或字符串检测无法根治 HTML 注入，因为解析器行为本身具有状态性和上下文敏感性。在整个安全体系中，它属于客户端注入攻击，与 CSP、SOP（同源策略）并列为前端安全的核心防线，专业工程师必须掌握它，因为现代前端框架（React/Vue）虽然默认转义，但仍存在 dangerouslySetInnerHTML、v-html 等逃逸口，且浏览器扩展、富文本编辑器、SVG 处理等场景直接操作 innerHTML 极为普遍；不理解 mXSS 的底层解析规则，就无法设计出真正抗变异的净化方案。

### 2. 底层原理剖析
底层运行机制核心在于 HTML 解析器的状态机与 DOM 序列化的不对称性。HTML5 解析规范定义了多种解析状态（如 data state、tag open state、RAWTEXT state、RCDATA state 等），不同标记（如 <script>、<textarea>、<title>）会切换解析状态，使得内部的文本不再按 HTML 标记解析。当调用 element.innerHTML = htmlString 时，浏览器将字符串喂给解析器，生成 DOM 树；当读取 element.innerHTML 时，浏览器调用 HTML 序列化算法（fragment serializing algorithm）将 DOM 树转换为字符串。mXSS 的关键在于：序列化结果可能不再等价于原始输入字符串，因为解析器会修复格式错误的标签、丢弃无效属性、归一化标签名等。例如输入 '<img src=x onerror=alert(1)>' 被直接设置时会执行，但若经服务端过滤掉 onerror 属性，可能留下 '<img src=x>'；此时若外层包裹 <noscript>，则序列化后 <noscript> 的内容会被当作纯文本处理，但再次解析时（比如二次赋值）<noscript> 进入 RAWTEXT 状态，其内部内容不再解析为 HTML，这是防护机制；然而攻击者可以利用 </noscript> 提前闭合，或者使用 <svg><p> 等混合命名空间使解析器产生新节点，导致序列化后的字符串在下次被插入到不同上下文时产生新语义。精确的伪代码流程：
1. 攻击者构造 payload：'<svg><style><img src=x onerror=alert(1)>'。
2. 首次赋值：div.innerHTML = payload；解析器解析 <svg> 进入 foreign content 模式，<style> 进入 RAWTEXT 状态，因此 '<img src=x onerror=alert(1)>' 被当作 CSS 文本，不执行。
3. 读取：div.innerHTML 序列化，因为 <style> 在 foreign content 中，序列化器输出 '<svg><style><img src=x onerror=alert(1)>'，看起来一致。
4. 但若外层有 <math> 或 <noscript> 等特殊元素，序列化器可能对某些字符（如 '>', '/'）进行转义或对注释符进行处理，导致在第三次解析时，<style> 结束位置改变，使 onerror 属性被激活。
更经典的 mXSS 利用模式：
- 输入被净化器转为 '<math><mtext><table><mglyph><style><!--</style><img title="--><img src=1 onerror=alert(1)>">'。净化器认为 <style> 内是注释，安全。但浏览器解析 <math> 后进入 foreign content，<mtext> 后进入 text 状态，<table> 会触发 foster parenting（移出文本），最终 DOM 结构变化，序列化后生成 '<img src=1 onerror=alert(1)>'，导致后续 innerHTML 注入执行。
与前端已有概念的对比：它类似于 Java 的接口与 TypeScript 接口的区别——Java 接口是运行时的实体，有字节码结构，强制实现；TS 接口仅存在于编译期，是类型系统的擦除。类比到 XSS：传统字符串过滤好比 TS 的静态检查，认为类型安全就运行时安全；而 mXSS 揭示的是运行时的 HTML 解析器（类似 Java 虚拟机）具有自己的语义规则，静态字符串看似安全（类型通过），但解析器运行时行为（字节码执行）会产生意外结果。这教导我们：对 HTML 的净化必须在解析后的 DOM 层面进行，而不是在字符串层面进行。

### 3. 基础代码与实战验证
以下为验证 mXSS 的基础代码，不依赖框架，直接在浏览器控制台或独立 HTML 文件中运行。

```html
<!DOCTYPE html>
<html>
<body>
<div id="container"></div>
<script>
// 步骤1：定义攻击 payload。注意：这里 payload 在字符串层面看似安全——
// 它包含一个 <style> 标签，内部文本看似为 CSS 注释，不包含可执行属性。
// 但该字符串在两次 innerHTML 赋值之间，会被浏览器解析-序列化-再解析，产生突变。
const payload = '<math><mtext><table><mglyph><style><!--</style><img title="--><img src=1 onerror=alert(1)>">';

// 步骤2：首次赋值给一个离屏元素（不插入文档树，避免立即执行）。
// 浏览器内部调用 HTML 解析器将 payload 解析为 DOM 树。
// 此时由于 foreign content（<math>）和特殊元素状态，<style> 内的文本被视为 RAWTEXT，
// 其中的 '<img src=1 onerror=alert(1)>' 被当作普通文本，不会执行。
const temp = document.createElement('div');
temp.innerHTML = payload;

// 步骤3：读取 innerHTML，触发 DOM 序列化算法。
// 序列化器根据当前 DOM 树结构重新生成 HTML 字符串。
// 由于解析期间的 foster parenting 和命名空间调整，DOM 树结构与原始字符串的语法树不同，
// 导致序列化出的字符串中，攻击者数据被移动到新的位置，且去掉了某些转义。
const mutated = temp.innerHTML;

// 步骤4：将突变后的字符串再次赋值给页面中的可见元素。
// 此时第二次解析会按照新的字符串生成 DOM，原 <style> 和注释结构被破坏，
// 攻击者控制的 <img onerror> 被激活为实际的事件属性，从而执行 alert。
document.getElementById('container').innerHTML = mutated;
// 如果执行成功，会弹出 alert(1)，证明 mXSS 发生。
</script>
</body>
</html>
```

关键代码行注释：
- `temp.innerHTML = payload`：首次解析，触发解析器状态机；`payload` 中的 `<math>` 使解析器进入 foreign content 模式，`<style>` 使内部内容进入 RAWTEXT 状态，所以 `<img src=1 onerror=...>` 此时不作为标签被识别。
- `const mutated = temp.innerHTML`：序列化器读取 DOM 树，由于 `table` 的 foster parenting 机制（`<table>` 会将其内部的非表格内容移出到 `<table>` 之前），原本位于 `<style>` 内部的部分内容被重新放置到 `<style>` 外部，同时 `<style>` 的闭合标签被序列化为 `</style>`，使得下次解析时 `onerror` 属性被识别。
- `document.getElementById('container').innerHTML = mutated`：第二次赋值，新的字符串不再包含保护性的 `<!--` 注释结构，`<img>` 变为真实标签，`onerror` 成为事件属性，触发脚本执行。

若无法在控制台直接触发（某些浏览器已修复该特定变体），可替换为现代变体如：`<svg></p><style><img src=x onerror=alert(1)>`，原理相同：利用 `</p>` 在 foreign content 中的解析异常使序列化结果改变。验证的核心不是具体 payload，而是观察 `mutated` 与 `payload` 的字符串差异。

### 4. 常见误区与进阶思考
误区一：认为对用户输入进行 HTML 实体转义（如 < 替换为 &lt;）或使用 sanitizer 库（如 DOMPurify）就能绝对防御 XSS。实际上，mXSS 正是利用了转义后的实体在特定解析上下文中的反转义行为。例如 DOMPurify 早期版本曾因 mXSS 被绕过，因为净化器在字符串层面处理，而浏览器在 DOM 层面解析。净化必须在真实的 DOM 解析流程之后，再对解析后的 DOM 进行清理，并且要考虑到序列化-再解析的幂等性。正确的做法是：使用 `Node.textContent` 代替 `innerHTML` 来设置文本；必须用 `innerHTML` 时，应当对 DOM 树进行遍历并移除所有事件属性、`javascript:` 协议 URL 和 `<script>` 节点，而不是依赖字符串匹配。

误区二：认为 CSP（内容安全策略）可以完全阻止 mXSS。CSP 可以限制脚本执行来源，但无法阻止属性注入或 HTML 元素注入（如 `<form action>`、`<meta>`、`<iframe>` 等可能导致的数据泄露或钓鱼）。此外，如果 CSP 中使用了 `unsafe-inline` 或者通过 DOM 注入 `<script>` 元素（虽然 CSP 会阻止加载，但某些浏览器对动态插入的 `<script>` 的 CSP 处理存在差异），仍然可能绕过。CSP 是纵深防御的一层，不能替代正确的 HTML 净化。

深度思考题：给定一个字符串 `s`，假设你写了一个净化函数 `sanitize(s)`，它先将 `s` 通过 `new DOMParser().parseFromString(s, 'text/html')` 解析，然后移除所有属性名为 `on*` 的属性和所有 `<script>` 标签，最后序列化返回 `result`。请问：如果再次将 `result` 赋值给某个元素的 `innerHTML`，是否绝对安全？如果不绝对安全，请描述一种攻击方式，并说明需要额外增加什么净化步骤才能确保安全。（提示：考虑 `parseFromString` 的解析上下文与 `innerHTML` 的解析上下文差异，以及某些浏览器对 `<noscript>`、`<template>` 的特殊处理。）
