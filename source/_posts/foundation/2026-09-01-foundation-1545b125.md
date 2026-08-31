---
title: "每日基础技术总结 · 2026-09-01 · 原型链与闭包"
date: 2026-09-01 07:20:30
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-01 · 原型链与闭包

## 📚 今日主题

> **原型链与闭包**（前端底层与计算机基础）

### 1. 核心概念速览
原型链（prototype chain）是 JavaScript 对象继承的底层机制，本质是基于对象的委托（delegation），而非基于类的拷贝继承。每个对象内部都有一个 [[Prototype]] 引用，指向其构造函数的 prototype 对象；当访问对象属性时，引擎沿 [[Prototype]] 链向上查找，直到 Object.prototype 的 [[Prototype]] 为 null 为止。闭包（closure）是函数与其词法环境（lexical environment）的组合，本质是函数在定义时捕获了其外层作用域中的变量绑定，并在函数被调用时仍能访问这些绑定，即使外层函数已返回。原型链解决的是对象间的属性共享与复用；闭包解决的是变量作用域的持久化与私有状态的封装。在计算机体系里，原型链对应基于委托的动态分派机制，闭包对应函数式语言中的环境捕获与高阶函数基础。专业工程师必须掌握，因为它们在 JS 运行时内存模型、模块化设计、前端框架源码、事件循环及异步编程中无处不在，且直接影响内存泄漏、性能优化与抽象边界的设计能力。

### 2. 底层原理剖析
当创建一个函数 F 时，JS 引擎会同时创建两个对象：函数对象 F 和原型对象 F.prototype。F.prototype 默认有一个 constructor 属性指回 F。执行 new F() 时，引擎创建新对象 obj，并将 obj 的内部槽 [[Prototype]] 指向 F.prototype。属性访问 obj.key 时，引擎先检查 obj 自身属性，若不存在，则沿 obj.[[Prototype]] 链逐层查找，直到找到属性或链尾 null。Object.prototype 的 [[Prototype]] 是 null，构成闭环。class 语法只是上述 [[Prototype]] 机制之上的语法糖，extends 实际设置构造函数的 prototype 链（用于静态继承）和实例的 [[Prototype]] 链（用于方法继承）。与 Java 的接口对比：Java 接口是编译期约束的类型契约，类必须显式 implements，运行时对象没有动态改变结构的能力；TS 的接口同样是编译期结构类型（duck typing），不产生任何运行时对象。而 JS 的原型链是运行时动态委托，任何对象可以在任意时刻修改其 prototype 指向（通过 Object.setPrototypeOf 或修改 prototype 上的属性），影响所有链接到该原型对象的实例。闭包的底层是 Lexical Environment 对象。函数定义时，引擎为函数保存一个 [[Environment]] 内部槽，指向当前执行上下文的 Outer Environment。函数调用时创建新的 Function Environment，其 outer 指向 [[Environment]]。因此，即使外层函数执行完毕，其 Environment 也不会被 GC 回收，只要闭包函数仍被引用。闭包捕获的是变量绑定（binding）而非值快照，所以循环中用 var 声明时，所有闭包共享同一变量绑定；let 每次迭代创建新绑定，因此闭包分别捕获不同环境。与 Java 的成员内部类对比：Java 内部类隐式持有外部类的 this 引用，本质是对象级关联；JS 闭包捕获的是变量环境，更接近 Java 方法内定义局部类时捕获 final 局部变量（Java 8 后 effectively final），但 JS 闭包捕获的变量可修改。

### 3. 基础代码与实战验证
```text
// 原型链验证
function Animal(name) {
  this.name = name; // 实例自身属性
}
Animal.prototype.speak = function() {
  return this.name + ' makes a sound';
};
const dog = new Animal('Rex');
// 当访问 dog.speak 时，dog 自身没有 speak，引擎沿 dog.[[Prototype]] 找到 Animal.prototype.speak
console.log(Object.getPrototypeOf(dog) === Animal.prototype); // true
// Animal.prototype 的 [[Prototype]] 指向 Object.prototype
console.log(Object.getPrototypeOf(Animal.prototype) === Object.prototype); // true
// Object.prototype 的 [[Prototype]] 为 null，链终止
console.log(Object.getPrototypeOf(Object.prototype) === null); // true

// 闭包验证
function createCounter() {
  let count = 0; // 被闭包捕获的变量绑定，存在于 createCounter 的环境对象中
  return function() {
    count += 1; // 每次调用更新同一个绑定
    return count;
  };
}
const counter = createCounter();
// createCounter 已返回，但其环境未释放，因为返回的函数仍持有对该环境的引用
console.log(counter()); // 1
console.log(counter()); // 2

// 循环闭包陷阱：var  vs  let
const funcs = [];
for (var i = 0; i < 3; i++) {
  funcs.push(function() { return i; }); // 捕获同一个外层变量 i
}
console.log(funcs[0]()); // 3，因为 i 是同一个绑定，最终变为 3
const funcs2 = [];
for (let j = 0; j < 3; j++) {
  funcs2.push(function() { return j; }); // let 每次迭代创建新绑定
}
console.log(funcs2[0]()); // 0，每个闭包捕获自己迭代的环境
```

### 4. 常见误区与进阶思考
误区一：认为闭包捕获的是变量的值，而不是变量绑定。真相是闭包捕获的是 binding（引用），所以外部变量的后续修改会反映在闭包内部，这也导致 for 循环 var 陷阱。正确理解需要从 Execution Context 和 Lexical Environment 的引用关系出发，不是简单的值复制。误区二：认为原型链是继承的替代品，因此可以随意修改 prototype。真相是原型链的动态性是双刃剑：修改 Array.prototype 会影响所有数组实例，且代码可读性和性能都有代价（V8 中隐藏类与内联缓存（IC）会被动态修改破坏）。真正理解原型链需要区分“自有属性”和“委托属性”，并清楚属性遮蔽（property shadowing）机制。

进阶思考题：给定如下代码，解释输出顺序，并说明每一步发生的作用域查找过程：
let x = 1;
function A() {
  let x = 2;
  function B() {
    console.log(x);
  }
  return B;
}
const C = A();
C(); // 输出多少？为什么？如果去掉 A 内部声明 x，输出会变化吗？这如何从 Lexical Environment 链的 outer 指向来解释？
