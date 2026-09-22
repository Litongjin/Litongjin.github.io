---
title: "每日基础技术总结 · 2026-09-15 · 原型链与闭包"
date: 2026-09-15 07:03:10
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-15 · 原型链与闭包

## 📚 今日主题

> **原型链与闭包**（前端底层与计算机基础）

### 1. 核心概念速览
## 核心概念速览

**原型链**：JavaScript 对象通过内部槽 `[[Prototype]]` 指向另一个对象（或 `null`），形成单向链表。属性访问时，引擎执行 `Get` 操作：先查自有属性，未命中则沿 `[[Prototype]]` 向上查找，直到 `null` 返回 `undefined`。本质是**对象间的动态属性委托**，而非复制继承。它解决的问题：在多个对象间共享属性/方法，实现继承，减少内存占用。机制：每个函数对象有 `prototype` 属性；`new F()` 创建的新对象，其 `[[Prototype]]` 指向 `F.prototype`；函数本身也是对象，其 `[[Prototype]]` 指向 `Function.prototype`。

**闭包**：函数与其定义时词法环境（Lexical Environment）的组合。本质是**函数对象持有对外部词法环境记录（Environment Record）的引用**，使得外部函数的局部变量在外部函数返回后仍可被访问。它解决的问题：状态私有化、数据封装、函数工厂、柯里化、回调保持上下文。机制：函数创建时，引擎将其内部槽 `[[Environment]]` 指向当前词法环境；当函数逃逸出定义它的函数，该环境从栈帧转移到堆（或本就堆分配），由 GC 管理。

**体系位置**：原型链是 JS 对象模型与属性查找的核心，决定继承、`this`、`instanceof`、`class` 语法糖的底层行为；闭包是函数式编程、作用域、内存管理的核心，决定模块模式、React Hooks 闭包陷阱、事件回调、异步状态保持。**为什么必须掌握**：它们直接影响代码运行时行为、内存占用、性能与调试；框架底层（如 React Fiber、Vue 响应式、Node 模块）均建立在这两者之上。

### 2. 底层原理剖析
## 底层原理剖析

### 原型链：动态查找与委托

伪代码描述 `new` 与属性查找：

    function New(F, args) {
      let obj = {};
      obj.[[Prototype]] = F.prototype;   // 设置内部槽，非自有属性
      let result = F.apply(obj, args);   // 以 obj 为 this 执行构造函数
      return (typeof result === 'object' && result !== null) ? result : obj;
    }

    function Get(obj, key) {
      let cur = obj;
      while (cur !== null) {
        if (HasOwnProperty(cur, key)) return cur[key];
        cur = cur.[[Prototype]];        // 沿链向上，运行时动态解析
      }
      return undefined;
    }

关键点：`[[Prototype]]` 是内部槽，规范中不可直接访问（ES6 后可通过 `Object.getPrototypeOf` / `Object.setPrototypeOf` 操作）。属性查找是**运行时**行为，原型可随时修改，已有实例立即受影响。赋值 `obj.key = v` 时，若 `key` 在原型链上且为可写数据属性，则在 `obj` 上创建自有属性遮蔽原型；若为访问器属性，则调用 setter。

### 闭包：词法环境与逃逸

伪代码描述闭包创建与调用：

    function Outer() {
      let x = 0;                         // 在 Outer 的词法环境中创建绑定 x
      function Inner() { x++; return x; } // Inner.[[Environment]] = Outer 的词法环境
      return Inner;
    }

    let fn = Outer(); // Outer 执行完毕，执行上下文出栈
    // 但 Inner 对象仍引用 Outer 的词法环境，该环境被提升/保留在堆中
    fn(); // 调用时创建新执行上下文，其外层词法环境 = Inner.[[Environment]]，读写 x

闭包捕获的是**变量绑定（binding）**，不是值快照。因此同一个环境中的多个闭包共享同一绑定；不同次调用 `Outer` 会产生不同环境，互不干扰。

### 与前端已有概念的对比

- **原型链 vs Java/C++ 类继承**：Java 类继承在编译期确定虚方法表（vtable），方法调用是固定偏移的虚表查找；JS 原型链是运行时动态查找，原型可动态修改，属性可被实例自有属性遮蔽。`class` 是语法糖，方法仍定义在 `prototype` 上。
- **原型链 vs TypeScript 接口**：TS 接口是编译期结构类型（structural typing），无运行时实体，仅用于类型检查；Java 接口是运行时名义类型（nominal typing），类通过 `implements` 声明，方法调用类似虚表。原型链不是接口，而是对象间属性委托机制。
- **闭包 vs Java Lambda 捕获**：Java Lambda 捕获的是 effectively final 变量的值副本（或要求 final），不可修改原变量；JS 闭包捕获变量绑定，可读写可变绑定，变量在堆中存活，由 GC 管理。C++ Lambda 按值/引用捕获，生命周期需显式管理；JS 由引擎自动管理。

### 3. 基础代码与实战验证
```text
## 基础代码与实战验证

    // 原型链验证
    function Foo() { this.own = 'own'; }
    Foo.prototype.protoProp = 'proto';
    const f = new Foo();
    console.log(f.own);        // 'own'：自有属性，直接命中，不查原型
    console.log(f.protoProp);  // 'proto'：自有属性不存在，沿 [[Prototype]] 找到 Foo.prototype
    console.log(Object.getPrototypeOf(f) === Foo.prototype); // true：new 时设置内部槽
    console.log(f.hasOwnProperty('protoProp')); // false：原型属性不是自有属性
    Foo.prototype.protoProp = 'changed';
    console.log(f.protoProp);  // 'changed'：动态查找，原型修改立即影响所有未遮蔽实例
    Object.setPrototypeOf(f, null);
    console.log(f.protoProp);  // undefined：切断原型链，查找终止于 null

    // 闭包验证：变量绑定存活于堆中
    function outer() {
      let count = 0;           // 局部变量，本应在 outer 返回后随栈帧销毁
      return function inner() {
        count++;               // 通过 inner.[[Environment]] 访问 outer 的词法环境
        return count;
      };
    }
    const counter = outer();   // outer 执行完毕，但 count 被 inner 闭包引用，环境被保留
    console.log(counter());    // 1：读取并修改同一绑定
    console.log(counter());    // 2：状态在多次调用间持久化
    const counter2 = outer();  // 新的 outer 调用，新的词法环境，独立 count
    console.log(counter2());   // 1：与 counter 互不干扰

    // 共享绑定验证：var vs let
    function createFns() {
      var fns = [];
      for (var i = 0; i < 3; i++) {
        fns.push(function() { return i; }); // 三个闭包共享同一个 i 绑定（函数作用域）
      }
      return fns;
    }
    console.log(createFns().map(fn => fn())); // [3, 3, 3]：循环结束后 i=3

    function createFnsLet() {
      const fns = [];
      for (let i = 0; i < 3; i++) {
        fns.push(function() { return i; }); // let 每次迭代创建新绑定，每个闭包捕获不同 i
      }
      return fns;
    }
    console.log(createFnsLet().map(fn => fn())); // [0, 1, 2]
```

### 4. 常见误区与进阶思考
## 常见误区与进阶思考

**误区 1：认为原型链是复制继承，或认为实例属性来自构造函数。** 实际上 `new` 只设置 `[[Prototype]]` 并执行构造函数；方法定义在 `prototype` 上，实例通过动态查找访问。在实例上赋值同名属性会创建自有属性遮蔽原型；修改原型会影响所有未遮蔽实例。

**误区 2：认为闭包捕获的是值，或认为 `var` 和 `let` 在闭包中行为相同。** 闭包捕获的是词法环境中的**变量绑定**。`var` 函数作用域内只有一个绑定，循环中所有闭包共享它；`let` 块作用域每次迭代创建新绑定，每个闭包捕获独立的 `i`。另一个相关误区是认为闭包必然内存泄漏：只要闭包可达，其引用的环境就保留；解除对闭包的引用后，环境可被 GC 回收。

**进阶思考题**：在 V8 中，当同一词法作用域内创建多个闭包时，它们如何共享同一个 `Context` 对象？为什么 `eval` 或 `with` 会导致该作用域的变量全部被堆分配而无法优化？请从词法环境、变量绑定与 GC 可达性角度解释。
