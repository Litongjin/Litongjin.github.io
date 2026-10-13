---
title: "每日基础技术总结 · 2026-10-14 · HTML 解析器的状态机机制：自关闭标签与隐式结束标签的处理逻辑"
date: 2026-10-14 07:15:57
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-14 · HTML 解析器的状态机机制：自关闭标签与隐式结束标签的处理逻辑

## 📚 今日主题

> **HTML 解析器的状态机机制：自关闭标签与隐式结束标签的处理逻辑**（前端底层与计算机基础）

### 1. 核心概念速览
HTML 解析器是事件驱动的状态机，由标记化器（tokenizer）和树构建器（tree builder）两个协同状态机组成。标记化器负责将字节流切分为 start tag token、end tag token、character token 等；树构建器根据当前插入模式（insertion mode）与已打开的 DOM 栈处理这些 token，最终生成 DOM 树。

自关闭标签（self-closing tag）是源码中形如 <span/> 的写法：tokenizer 会为 start tag token 设置 self-closing flag，但 HTML 树构建阶段仅在外来内容（foreign content，如 SVG/MathML 子树）中承认该标志；对 HTML 命名空间内的非 void 元素（div、span、p 等），该标志被忽略并产生 parse error，元素保持打开，直到被显式或隐式闭合。

隐式结束标签（implied end tags）是树构建器为提高容错性而定义的自动闭合规则：当特定开始标签或结束标签出现时，即使源码未写出对应的 </tag>，解析器也会合成结束标签 token 并弹出栈上相应元素。典型用例包括 <li>、<p>、<option>、<tr>、<td> 等。

该机制解决的问题是真实世界的 HTML 大量省略闭合标签、嵌套不规范，解析器必须生成稳定的 DOM 结构。它位于浏览器渲染引擎的解析阶段，是后续 DOM API、CSS 选择器、脚本执行、布局计算的底层基础。专业工程师必须掌握，因为任何通过 innerHTML、document.write、DOMParser 注入或传输的 HTML 都经过同一状态机，不理解会导致 XSS、DOM 结构意外、脚本执行顺序失控等底层错误。

### 2. 底层原理剖析
HTML 解析使用两个耦合状态机：标记化状态机（tokenizer）和树构建状态机（tree builder）。标记化器按字符读取输入，在 data state、tag open state、tag name state、self-closing start tag state 等状态间转换，生成各类 token；其中自闭合标志在 tokenizer 遇到 '/' 紧跟 '>' 进入 self-closing start tag state 时被设置。树构建器维护一个 DOM 栈（stack of open elements），根据当前 insertion mode 处理 token；insertion mode 是树构建状态机的核心状态，如 "in body"、"in table"、"in select"、"in foreign content" 等。

自关闭标签处理逻辑：
处理 start tag token（self-closing flag = true）时：
  if 当前 insertion mode 属于 foreign content（且不在 HTML integration point）：
      承认 self-closing flag，插入元素后立即弹出，元素不进入长期打开状态
  else if 元素属于 HTML void elements（如 br、img、input）：
      元素本身无子节点，自闭合标志无实际影响
  else：
      忽略 self-closing flag，记录 parse error，按普通 start tag 插入 DOM 栈
      元素保持打开，后续 token 成为其子节点，直到遇到能闭合它的 end tag 或隐式结束规则触发

隐式结束标签处理逻辑：
HTML 标准为许多元素定义了 end tag omission 规则以及 start tag 触发的闭合规则。例如在 "in body" insertion mode 中：
  - 遇到 start tag <li>：如果栈中已存在一个 li 元素且当前节点是 li 元素，则生成该 li 的结束标签并弹出到 li 为止，然后插入新 li；否则直接插入。
  - 遇到 start tag <p>：如果栈中已存在一个 p 元素在 scope 中，则生成 p 的结束标签并弹出到 p 为止；然后再插入新 p。
  - 遇到 end tag </p>：如果栈中找不到 p 元素，则合成一个 p 开始标签并立即闭合，以保证文档结构不产生孤立结束标签。
  - 表格相关插入模式会强制结束当前 td/th/tr 等。

这些规则本质上是树构建器的确定性状态转移函数：输入 (当前 insertion mode, 当前栈, token)，输出 (新 insertion mode, 修改后的栈, 生成的 token 或错误)。

与前端已有概念对比：XML 解析器是确定性的无容错状态机，一旦自闭合标志出现则严格生成空元素；而 HTML 解析器是容错的、带有大量恢复策略的状态机，自闭合仅在外来内容中有效。类似 JavaScript 的自动分号插入（ASI）是词法/语法级的容错机制，HTML 的隐式结束标签是树构建级的容错机制，两者都通过隐式合成规则将不规范源码映射到规范结构，但 ASI 作用在 token 流，而隐式结束标签作用在 DOM 树构建阶段。

### 3. 基础代码与实战验证
```text
// 运行环境：现代浏览器 DevTools Console
const html = `<div><span/>text</div><ul><li>one<li>two</ul>`;
const doc = new DOMParser().parseFromString(html, 'text/html');

// 1. 自闭合标志被 HTML 树构建器忽略：
//    tokenizer 已将 <span/> 标记 self-closing flag，但 span 非 void 元素且不在 foreign content，
//    flag 被丢弃，span 保持打开；后续字符 token "text" 被插入 span 子节点，
//    span 直到遇到 </div> 的隐式闭合机制才被弹出。
const span = doc.querySelector('span');
console.log(span.childNodes.length); // 1（文本节点）
console.log(span.textContent);       // 'text'
console.log(span.parentElement.tagName); // 'DIV'

// 2. 隐式结束标签：<li> 的 start tag 触发前一个 li 的隐式 end tag，
//    因此第二个 li 不是第一个 li 的子元素，而是兄弟节点。
const listItems = [...doc.querySelectorAll('ul > li')];
console.log(listItems.length);                    // 2
console.log(listItems[0].nextElementSibling === listItems[1]); // true
console.log(listItems[1].childElementCount);      // 0，证明未被嵌套

// 序列化结果佐证（现代浏览器一致）：
console.log(doc.body.innerHTML);
// 输出类似：<div><span>text</span></div><ul><li>one</li><li>two</li></ul>
```

### 4. 常见误区与进阶思考
常见误区一：把 XML/XHTML 的自闭合规则套用到 HTML。很多工程师以为 <script src="a.js"/> 会生成自闭合脚本元素。实际上在 HTML 中 script 不是 void 元素，也不处于 foreign content，自闭合标志被忽略，导致后续内容全部被吞入 script 的文本直到真正的 </script>，可能造成脚本注入或 DOM 异常。必须使用显式 </script>。

常见误区二：认为所有开始标签都会闭合前一个同名元素或块级元素。实际上隐式结束规则是白名单制，只对标准明确列出的标签触发；例如 <span>a<span>b 产生嵌套 span，而非兄弟。对于不熟悉的标签，应查标准或实测 DOM 结构，不能依赖直觉。

进阶思考题：在 HTML 源码 <svg><circle/></svg> 与 <div><span/></div> 中，自闭合标志的处理为何不同？请从 tree builder 的 insertion mode（foreign content vs in body）与元素命名空间出发，说明 circle 为何是自闭合，而 span 不是；并进一步推断若在 SVG 子树内出现 HTML integration point（如 <foreignObject>），规则又会如何变化。
