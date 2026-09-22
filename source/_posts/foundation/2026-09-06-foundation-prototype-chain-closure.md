---
title: "每日基础技术总结 · 2026-09-06 · 原型链与闭包"
date: 2026-09-06 07:01:43
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-06 · 原型链与闭包

## 📚 今日主题

> **原型链与闭包**（前端底层与计算机基础）

### 1. 核心概念速览
原型链是 JavaScript 对象间属性/方法委托的运行时机制：每个对象内部都有一个 [[Prototype]] 槽位指向另一个对象，当属性访问在当前对象找不到时，沿着该链向上查找直到 Object.prototype 或 null。闭包是词法作用域与一等函数结合的机制：函数在定义时通过内部 [[Environment]] 槽捕获其外层 Lexical Environment，使它在被调用时（无论在哪）依然能访问定义时可见的那些绑定。二者解决的核心问题分别是：在没有类继承语法（原始 JS）的情况下实现行为复用，以及在函数作为值传递时维持私有状态/回调上下文。该知识点属于 ECMAScript 规范与执行引擎层面的基础，也是事件循环、异步模型、函数式组合等一切上层抽象的地基。专业工程师必须掌握，因为原型链影响属性访问性能与内存结构，闭包影响变量生命周期与内存回收，错误的理解会造成性能退化、内存泄漏和跨模块状态污染。

### 2. 底层原理剖析
一、原型链本质：JS 的属性访问使用 GetValue 语义。设对象为 O，访问属性 P：先 O.[[GetOwnProperty]]，若不存在则取 O.[[Prototype]] 指向的对象继续同样过程，直到 null 返回 undefined。对象字面量 {} 的 [[Prototype]] 默认指向 Object.prototype；Object.create(proto) 显式指定；Object.setPrototypeOf 可修改；__proto__ 是访问器属性。构造函数的 prototype 属性只是 new 运算时赋给新实例 [[Prototype]] 的母本，并非函数自身的原型（函数自身原型是 Function.prototype）。class 语法只是该机制语法糖，extends 建立 [[Prototype]] 委托。
二、闭包本质：JS 执行上下文由 ExecutionContext 的 LexicalEnvironment 记录变量绑定，函数对象除代码外还有 [[Environment]] 槽，指向函数定义时当前作用域的环境记录。调用时，创建新 Function Environment，其 outer 引用等于 [[Environment]]。因此内层函数引用外层变量时，引擎沿 outer 链解析。闭包不是“快照”，而是绑定（binding/cell）的保持，因此外部变量变化时闭包看到最新值。
三、与前端已有概念的对比：TS interface 是纯编译期结构约束，编译后不生成任何代码；原型链是运行时真实存在的指针链，interface 不能用 instanceof 检测（值层面不存在）。Java 接口是类型系统契约，需类显式实现；JS 原型是对象间隐式委托，不需要声明实现，运行时属性存在即可。Java 的匿名内部类捕获局部变量必须是 final 的（值快照）；JS 闭包捕获的是可变绑定（引用环境），所以可以自由修改并与外部同步，这正是横向对比时的核心差异。

### 3. 基础代码与实战验证
```text
// 验证闭包：函数捕获的是变量绑定，不是值
function createCounter() {
  let count = 0; // 本地环境记录中的绑定
  return function() {
    count++; // 解析时沿 outer 找到 createCounter 环境的 count 绑定
    return count;
  };
}
const c1 = createCounter();
c1(); // 1
c1(); // 2
const c2 = createCounter(); // 新环境，自己的 count
c2(); // 1

// 验证原型链：属性查找沿 [[Prototype]] 向上委托
const proto = {
  greet() { return 'hello ' + this.name; }
};
const obj = Object.create(proto); // obj.[[Prototype]] 指向 proto
obj.name = 'js';
console.log(obj.greet()); // 无自有 greet，沿链找到 proto.greet，this 绑定 obj

// 切断原型链隔离属性
const empty = Object.create(null);
console.log(empty.toString); // undefined：不再委托到 Object.prototype

// 验证闭包中的更新可见性
let x = 1;
const readX = () => x; // [[Environment]] 绑定当前环境
x = 2;
console.log(readX()); // 2，证明捕获的是绑定而非快照
```

### 4. 常见误区与进阶思考
误区1：混淆函数的 prototype 属性与实例的 [[Prototype]]。普通函数有 prototype 属性，它仅用于 new 创建对象时作为新对象的 [[Prototype]]；函数自身继承自 Function.prototype。箭头函数和对象方法没有 prototype（不可作为构造器），但它们仍有 [[Prototype]]。
误区2：认为闭包导致内存泄漏是闭包的必然问题。闭包不会主动泄漏，只要闭包引用的环境不再被可达引用持有，GC 即可回收；“泄漏”只发生在开发者把长期存活结构（如全局缓存）挂到闭包所引用环境上时，才使该环境及其闭包变量无法释放。真正的理解是：闭包延长了变量生命周期，延长有代价；是否泄漏取决于引用图的可达性。
思考题：let o = { v: 1 }; const f = () => o; o = null; 请问 f() 返回值是什么？如果你认为闭包捕获的是“值”会答错；正确推理是：闭包捕获了环境记录中的绑定 o，赋值 o = null 修改了该绑定，所以 f() 返回 null。若想让闭包始终拿到原对象，需要在外部变量被改写前把对象存入一个不可变绑定（如 const）中，但那仍捕获绑定而非值。
