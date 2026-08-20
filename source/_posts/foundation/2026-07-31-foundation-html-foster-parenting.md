---
title: "每日基础技术总结 · 2026-07-31 · HTML 解析器中的标记化与树构建（含 foster parenting）"
date: 2026-07-31 08:00:00
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-31 · HTML 解析器中的标记化与树构建（含 foster parenting）

## 📚 今日主题

> **HTML 解析器中的标记化与树构建（含 foster parenting）**（前端底层与计算机基础）

### 1. 核心概念速览
HTML 解析器并非'字符串转 DOM'的简单工具，而是 WHATWG HTML Standard 规定的两阶段算法：标记化（tokenization）与树构建（tree construction）。标记化是把已解码的字符流转换为 token 流的有穷状态机；树构建则按当前 insertion mode 消费 token，维护 open elements 栈，并可能反向改变 tokenizer 状态（例如插入 title/textarea 后切换到 RCDATA/RAWTEXT）。foster parenting 是树构建阶段针对表格内非法嵌套的容错机制：当在 in table 上下文中遇到需要交给 in body 处理的 token 时，将插入点重定向到表格外，避免生成 table 内直接嵌套 div/p 这类非法 DOM，同时保持内容顺序。它位于网络字节流与 DOM/CSSOM 之间，是浏览器前端底层的解析管线。专业工程师必须掌握，因为 innerHTML、DOMParser、HTML 语义、XSS 防护和跨端 HTML 解析兼容性都建立在这套规范之上。

### 2. 底层原理剖析
标记化的本质是字符级状态机，不是正则。输入先经编码检测、BOM 剥离、CR/LF 归一化，然后逐字符驱动状态迁移。核心状态包括 Data、TagOpen、EndTagOpen、TagName、BeforeAttributeName、AttributeName、BeforeAttributeValue、AttributeValue（双引号/单引号/无引号）、SelfClosingStartTag、BogusComment、MarkupDeclarationOpen、Comment、DOCTYPE、CDATA、RCDATA、RAWTEXT、ScriptData 等。例如 Data 状态遇到 '<' 进入 TagOpen，TagOpen 遇到 '!' 进入 MarkupDeclarationOpen，遇到 '/' 进入 EndTagOpen。tokenizer 输出 DOCTYPE、StartTag、EndTag、Comment、Character、EOF 六类 token。注意 tokenizer 与树构建不是单向管线：树构建插入 title、textarea、script 后会设置 tokenizer 状态为 RCDATA、RAWTEXT、ScriptData，直到对应结束标签出现。

树构建的核心是 open elements 栈（栈顶即当前节点）与一组 insertion mode。每个 insertion mode 定义 token 的处理规则：initial、before html、before head、in head、after head、in body、text、in table、in table body、in row、in cell、in select、in template 等。插入新节点前先计算 appropriate place for inserting a node；只有启用 foster parenting 且目标为 table/tbody/tfoot/thead/tr 时，插入点才可能脱离当前栈顶。

foster parenting 的触发：在 in table 模式中，遇到不能由 in table 规则直接处理、需要交给 in body 处理的 token（如 div 的 StartTag、普通 Character token），规范先标记 parse error，启用 foster parenting，然后按 in body 规则处理该 token，随后禁用。插入点计算顺序：1. 若 open elements 栈中存在 template，插入 template 内；2. 否则找最后一个 table；3. 若没有 table，插入根 html 内；4. 若 table 有父节点，foster parent 为 table 的父节点，否则为 table 在栈中前一个元素；5. 若 table 有前一个元素兄弟，插入到该兄弟元素内部末尾；6. 否则插入到 foster parent 内、table 之前。因此 `<table><div>leak</div><tr><td>cell</td></tr></table>` 的 div 会成为 table 的前一个兄弟，而 tr/td 仍按正常表格路径进入 table。

与前端已有知识对比：tokenizer 相当于 Babel/TypeScript 的 scanner/lexer，树构建相当于 parser；但 HTML 没有形式化 CFG，只有规范算法加错误恢复。这类似于 Java 接口与 TypeScript 接口的差异：名称相同，但 Java 接口是名义类型系统下的编译期契约，TS 接口是结构化类型系统下的形状描述；同样，HTML parse 与 XML parse 都叫 parse，一个是容错状态机，一个是严格树约束，绝不能混用。

### 3. 基础代码与实战验证
```text
// 1) 用浏览器真实 HTML 解析器验证 foster parenting
const html = '<table><div>leak</div><tr><td>cell</td></tr></table>';
const doc = new DOMParser().parseFromString(html, 'text/html');
const table = doc.querySelector('table');
// div 不是 in table 模式允许的 token，树构建启用 foster parenting，将其插入 table 之前
console.assert(table.previousElementSibling.tagName === 'DIV');
console.assert(table.previousElementSibling.textContent === 'leak');
// tr/td 走正常表格路径，仍在 table 内
console.assert(table.querySelector('tr td').textContent === 'cell');

// 2) 插入点算法的极简伪代码（省略 template 分支）
// stack 是 open elements 栈，栈顶在数组末尾
function adjustedInsertionLocation(stack, fosterParentingEnabled) {
  if (!fosterParentingEnabled) {
    return { parent: stack[stack.length - 1], before: null };
  }
  const lastTable = [...stack].reverse().find(n => n.localName === 'table');
  if (!lastTable) {
    return { parent: stack[0], before: null };
  }
  if (lastTable.previousElementSibling) {
    // 有前一个元素兄弟时，插入到该兄弟元素内部末尾
    return { parent: lastTable.previousElementSibling, before: null };
  }
  // 无前一个元素兄弟时，插入到 table 的父节点中、table 之前
  return { parent: lastTable.parentNode, before: lastTable };
}
```

### 4. 常见误区与进阶思考
误区一：把 HTML 解析当作'正则或字符串替换'。tokenizer 是有穷状态机，树构建还要维护 open elements 栈和 insertion mode，并包含 adoption agency、foster parenting 等算法；正则只能做词法级匹配，无法表达上下文相关和错误恢复，因此任何'正则解析 HTML'的方案在真实 HTML 面前都不完备。误区二：把 foster parenting 理解为'先插入再移动节点'。实际流程是在插入前重定向 insertion location，DOM 从未出现过 table 内部包含 div 的中间状态；所以在 DOM 树中观察到的结果就是 div 天然在 table 外。思考题：为什么规范规定，若 table 有前一个元素兄弟，foster 节点要插入到该兄弟元素内部末尾，而不是直接插入到 table 之前？结合 insertion location 与 open elements 栈的关系，说明这种选择的必要性与代价。
