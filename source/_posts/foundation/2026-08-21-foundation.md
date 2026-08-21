---
title: "每日基础技术总结 · 2026-08-21 · 原型链与闭包"
date: 2026-08-21 17:41:49
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-21 · 原型链与闭包

## 📚 今日主题

> **原型链与闭包**（前端底层与计算机基础）

### 1. 核心概念速览
原型链（Prototype Chain）是 JavaScript 实现对象继承与属性解析的底层机制。每个对象内部都有一个[[Prototype]]引用（可通过__proto__访问，标准方式为Object.getPrototypeOf），指向其构造函数的prototype对象；当访问对象属性时，引擎先查找自身属性，若未找到则沿[[Prototype]]链逐级向上查找，直至Object.prototype，再往上为null。闭包（Closure）是词法作用域（Lexical Scope）与函数对象生命周期结合的产物：当函数在定义时的词法环境（Lexical Environment）中引用了外部变量，且该函数作为值被传递或返回时，JavaScript 引擎会保留该词法环境，使得函数在脱离定义作用域后仍能访问那些变量。闭包的本质是‘函数 + 其定义时捕获的环境记录（Environment Record）’。原型链解决的是属性/方法的复用与查找路径，闭包解决的是变量状态的持久化与数据私有化。在计算机体系里，原型链相当于一种基于委托的动态分派机制，类似自举式的方法解析；闭包则对应高阶函数与捕获语义，是函数式编程的基石。专业工程师必须掌握，因为两者是 JS 运行时行为、内存管理、性能优化（如隐藏类、内联缓存）和错误调试的根源，也是理解 TS 类型系统、React Hooks 依赖捕获、EventEmitter、模块化隔离等上层设计的底层前提。

### 2. 底层原理剖析
底层机制：原型链。JS 的每个函数在创建时都被赋予一个prototype属性，它是一个普通对象，内含constructor指向函数自身。使用new调用函数时，引擎会创建一个新对象，并把这个对象的[[Prototype]]指向该函数的prototype对象。属性访问（obj.key）执行[[Get]]：先查OwnPropertyKeys，若命中返回；否则取obj.[[Prototype]]，递归执行[[Get]]。该链的终点是Object.prototype（其[[Prototype]]为null）。因此，所有普通对象都能访问Object.prototype上的toString、hasOwnProperty等。注意：原型链是对象与对象之间的委托关系，而非类与类之间的复制关系。与 TS 接口对比：TS 接口是编译期结构类型（Structural Typing），用于约束形状，运行时不存在；原型链是运行时的实际对象关系，决定方法解析结果。Java 接口是编译期契约，类必须显式实现（implements），而 JS 原型链允许对象动态继承任意对象的方法，无需提前声明，且可动态修改（但现代引擎对动态修改有优化代价）。闭包底层机制：JS 使用词法环境（Lexical Environment）来管理变量绑定。每个函数有一个[[Environment]]内部槽，保存函数定义时所在的环境记录（由外层环境引用链构成）。函数执行时，引擎创建新的环境记录，其outer指向[[Environment]]。当函数返回后，若其环境记录仍被某个活动函数引用（即闭包），则该记录不会随调用栈销毁，而是存留在堆中。闭包捕获的是变量的引用而非值——这意味着多个闭包共享同一变量，且循环中var声明会产生共享问题，而let/const每次迭代创建新绑定。与前端已有概念对比：React Hooks 的useState/useEffect 依赖数组本质上是闭包捕获变量的快照对比；Vue 的响应式系统也是通过闭包收集依赖；模块化（ESM/CJS）导出的是闭包中的绑定，实现私有状态。JS 闭包等价于 Python 的nonlocal、C# 的lambda捕获、C++ 的lambda [=]（按引用捕获时），但 JS 没有显式声明捕获列表，完全由词法作用域隐式决定。注意闭包不是‘函数包函数’的语法现象，而是函数与环境的关联；即使不嵌套，只要函数被定义在某个作用域内并逃逸，也会形成闭包。

### 3. 基础代码与实战验证
```text
// 原型链验证
function Parent() {}
Parent.prototype.shared = function() { return 'from prototype'; };

const child = new Parent();
// 引擎执行：1. 创建空对象 obj；2. 设置 obj.[[Prototype]] = Parent.prototype；
// 3. 调用 Parent 作为构造函数（this=obj）；4. 返回 obj（若函数未显式返回对象）。

// 访问 child.shared：child 自身无 'shared' 属性，
// 引擎沿 [[Prototype]] 链查找到 Parent.prototype.shared，调用时 this 仍为 child。
console.log(child.shared()); // 'from prototype'

// 验证 child 与 Parent.prototype 是委托关系，而非复制：
console.log(child.hasOwnProperty('shared')); // false，自身属性中不存在
console.log(Object.getPrototypeOf(child) === Parent.prototype); // true

// 原型链的终点：
console.log(Object.getPrototypeOf(Parent.prototype) === Object.prototype); // true
console.log(Object.getPrototypeOf(Object.prototype) === null); // true

// 闭包验证：函数捕获环境记录，变量持久化
function createCounter() {
  let count = 0; // 该变量绑定在 createCounter 执行时的环境记录中
  return function increment() {
    // 该函数的 [[Environment]] 指向 createCounter 的环境记录
    count += 1; // 读取并修改捕获的绑定，不是副本
    return count;
  };
}

const counter = createCounter();
// createCounter 已返回，但其环境记录仍被 increment 引用，故不销毁。
counter(); // 1
counter(); // 2

// 验证闭包捕获的是引用而非值：
const fs = [];
for (var i = 0; i < 3; i++) {
  fs.push(function() { return i; }); // 三个函数共享同一个 i 绑定（var 提升）
}
console.log(fs.map(f => f())); // [3, 3, 3]
// 若改用 let，每次迭代创建新的绑定，则输出 [0, 1, 2]；
// 这是词法环境迭代快照与共享绑定差异的直观体现。
```

### 4. 常见误区与进阶思考
误区1：认为原型链是‘继承链’，并且修改一个对象的方法会影响所有后续实例——实际上原型链是‘委托链’，修改的是原型对象，实例自身不拷贝方法。实例上添加同名属性会遮蔽（shadow）原型属性，但不会修改原型；同理，给原型添加方法会立即影响所有现有实例（因为查找发生在运行时）。另一个极端误区是认为‘一切皆对象’导致认为所有对象都继承自 Function.prototype——实际所有普通对象的[[Prototype]]最终指向 Object.prototype，而 Function 自身是函数对象，其[[Prototype]]指向 Function.prototype，这是两条不同的链。误区2：认为闭包只是‘函数内部函数’的语法糖，或认为闭包一定会导致内存泄漏。闭包的本质是环境引用，只有当闭包本身仍被引用时环境才存活；若闭包被释放，环境随之可被 GC 回收。真正的问题是意外地让闭包长期持有大型外部变量（如未使用的 DOM 引用或大数据），从而延长生命周期。此外，误以为闭包捕获的是值，导致在循环中期望输出 0,1,2 而得到共享值——这是对词法环境绑定与执行时赋值时机理解不深。深度思考题：给定以下代码，解释输出并说明为何：
let a = 1;
function outer() {
  let a = 2;
  function inner() {
    console.log(a);
  }
  return inner;
}
const fn = outer();
const obj = { a: 3, fn };
obj.fn(); // 输出什么？请从词法环境解析顺序（由内到外，忽略调用者对象）说明为什么 JS 闭包不是动态作用域。
