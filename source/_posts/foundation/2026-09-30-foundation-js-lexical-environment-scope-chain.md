---
title: "每日基础技术总结 · 2026-09-30 · JavaScript 词法环境（Lexical Environment）与作用域链的底层结构"
date: 2026-09-30 07:05:48
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-30 · JavaScript 词法环境（Lexical Environment）与作用域链的底层结构

## 📚 今日主题

> **JavaScript 词法环境（Lexical Environment）与作用域链的底层结构**（前端底层与计算机基础）

### 1. 核心概念速览
词法环境（Lexical Environment）是 ECMAScript 规范中用于管理标识符绑定（Identifier Binding）的内部结构，本质是一个环境记录（Environment Record）+ 一个指向外部词法环境的引用（outer）。作用域链则是由当前执行上下文中的词法环境通过 outer 引用链向上逐级连接形成的链式结构，用于在变量解析（Identifier Resolution）时按从内到外的顺序查找绑定。它解决的核心问题是：在静态（词法）层面确定标识符的可见性与归属，使函数能够访问定义时的外层变量，而不是调用时。该机制是整个 JavaScript 闭包、块级作用域、this 绑定（非词法）的底层基石。在计算机体系中，它属于静态作用域（Lexical Scoping）的实现细节，与编译器/解释器中的符号表、作用域抽象相对应。专业工程师必须掌握它，因为变量提升、循环闭包陷阱、TDZ、eval/with 的影响、模块作用域等行为都可以由词法环境的创建与链接精确推导，也是理解调试器 Scope 面板、性能优化和编写正确高阶函数的前提。

### 2. 底层原理剖析
每个 JavaScript 执行上下文（Execution Context）在创建阶段都会关联一个词法环境。词法环境由两部分组成：环境记录（Environment Record）存储当前作用域内的标识符绑定（变量、函数、let/const/class 等）；outer 引用指向外部词法环境。函数创建时，会保存一个内部属性 [[Environment]]，指向函数定义时所在的作用域的词法环境。函数调用时，新的函数环境记录会创建，其 outer 被设置为该 [[Environment]]，而不是调用者的环境。作用域链实际上就是通过这条 outer 链完成的：当在当前环境记录中找不到某个标识符时，引擎沿着 outer 逐级向上查找，直到全局环境（global environment），全局环境的 outer 为 null。

关键机制：var 声明和函数声明存储在变量环境记录（VariableEnvironment）中，let/const/class 存储在词法环境记录（LexicalEnvironment）中；在 ES6 后两者可以分离，但对外统一通过词法环境访问。块级作用域（如 if/for 块）会创建新的词法环境，其外层是包含该块的环境；for 循环头部的 let 每次迭代都会创建新的词法环境并绑定当前迭代值，这是闭包捕获循环变量正确值的底层原因。函数声明在块级作用域中的行为按严格/非严格模式存在差异（Annex B 兼容）。

与前端已有概念的对比：类似 Java 的静态作用域但动态绑定实现不同；和 TS 的接口无直接关系——TS 接口是纯编译期类型抽象，而词法环境是运行时内存结构。更贴近的对比是 C/C++ 的栈帧与作用域：但 JS 的闭包使环境记录可以脱离调用栈存活（由垃圾回收决定生命周期），因此不是简单的栈帧，而是堆上可达的持存对象。与 Python/Common Lisp 的闭包类似，但 JS 的 var 提升与 let 的 TDZ 语义是独特细节。

### 3. 基础代码与实战验证
```text
function outer() {
  let x = 1;            // 进入 outer 调用时创建词法环境 LE1，x 绑定在 LE1 的环境记录中
  function inner() {    // 函数对象创建时保存 [[Environment]] = LE1
    return x;           // 解析 x：先在 inner 自己的环境记录找，找不到沿 outer 到 LE1，找到 x
  }
  return inner;
}

const fn = outer();    // outer 执行完成，LE1 未被释放，因为 fn 的 [[Environment]] 指向 LE1
console.log(fn());     // 1

// 块级作用域与 let 捕获
let funcs = [];
for (let i = 0; i < 3; i++) {   // 每次迭代创建新的词法环境 LE_i，i 绑定在 LE_i 中
  funcs.push(function() { return i; }); // 每个函数 [[Environment]] 分别指向 LE_0/LE_1/LE_2
}
console.log(funcs[0](), funcs[1](), funcs[2]()); // 0 1 2，而不是 3 3 3

// 如果用 var：
// for (var j = 0; j < 3; j++) { funcs.push(function(){ return j; }); }
// 所有函数共享全局/函数作用域中的同一个 j 绑定，输出 3 3 3
```

### 4. 常见误区与进阶思考
误区 1：认为作用域链是函数调用时的调用栈链。实际上它是函数定义时的词法嵌套链，由 [[Environment]] 决定，与调用者无关。

误区 2：认为 let/const 没有提升。其实 let/const 声明已被提升到所在作用域顶部并创建绑定，但在初始化执行之前处于不可访问的临时性死区（TDZ），访问会抛 ReferenceError——这不是“未声明”，而是“已绑定但未初始化”。

思考题：以下代码输出什么？请用词法环境的建立与查分机制解释：
let a = 1;
function f() {
  console.log(a);
  let a = 2;
}
f();
（答案：ReferenceError，因为进入函数时创建的环境记录已登记 a，但在执行到 let a = 2 前不可访问。）
