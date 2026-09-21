---
title: "每日基础技术总结 · 2025-04-05 · 原型链查找与 class 语法糖的本质"
date: 2025-04-05 20:00:00
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础（扩展）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2025-04-05 · 原型链查找与 class 语法糖的本质

## 📚 今日主题

> **原型链查找与 class 语法糖的本质**（前端底层与计算机基础（扩展））

### 1. 核心概念速览
原型链查找是 JavaScript 对象属性读取的底层解析机制，本质是基于 [[Prototype]] 内部槽位的线性遍历过程；Class 语法糖则是 ES6 引入的类式声明结构，其运行时完全基于构造函数（Constructor）与原型对象（Prototype Object）的组合模拟。该知识点处于 JS 引擎执行上下文的核心，是理解对象复用、继承及内存模型的基础。对于全栈工程师而言，掌握此机制能穿透语言表象，直接控制对象的属性分发策略与性能路径，避免在不必要的原型污染或隐式类型转换中产生逻辑漏洞，为后续深入 Node.js 源码或后端服务治理中的对象序列化/反序列化机制奠定理论基础。

### 2. 底层原理剖析
1. 实例创建：当使用 new Target() 时，JS 引擎执行以下步骤：
   - 创建一个全新的空对象 {}
   - 将该新对象的原型（[[Prototype]]）指向 Target.prototype
   - 将 this 绑定到新对象并执行 Target 函数体
   - 返回值（默认返回 this）
2. 原型链查找：obj.prop 读取操作触发内部方法 [[Get]]。
   - 步骤 A：检查 obj 自身是否存在 prop 属性。若存在且可枚举，返回其值。
   - 步骤 B：若不存在，则沿 [[Prototype]] 指针跳转至 prototypeObj。
   - 步骤 C：重复步骤 A-B，直到找到属性或到达 null（即 Object.prototype.[[Prototype]]）。此为短路逻辑，仅匹配第一个命中项。
3. Class 语法映射：
   - class Body { method() {} } 编译后等价于 function Body() {}
     Object.defineProperty(Body.prototype, 'method', { value: fn, enumerable: false, writable: true, configurable: true })
   - static method() {} 直接挂载在 Function Constructor 上，属于静态成员，不参与原型链实例查找。
4. 对比 TS/Java Interface：TS Interface 和 Java Interface 均为编译期静态类型检查约束，不生成任何运行时代码；而 JS Class 和 Prototype 是纯粹的运行时对象结构，决定内存布局和属性访问路径。

### 3. 基础代码与实战验证
```text
// 验证原型链查找与 Class 编译实质
function Base() {
  this.baseProp = 'base';
}
Base.prototype.getBase = function() { return this.baseProp; }; // 挂载到原型

class Derived extends Base {
  constructor() {
    super(); // 调用 Base.call(this)
    this.derivedProp = 'derived';
  }
  getDerived() { return this.derivedProp; }
  getBase() { return 'override'; } // 自身属性遮蔽原型属性
}

const instance = new Derived();

// 1. 自身属性优先：instance.getBase 在 Derived.prototype (instance) 中找到，返回 'override'
console.log(instance.getBase()); 

// 2. 原型链查找：instance.hasOwnBase 不在 Derived.prototype 中，
//    沿 __proto__ 指向 Base.prototype 查找，返回 undefined (Base.prototype 无此方法)
console.log(instance.hasOwnBase); 

// 3. 静态成员不通过原型链查找：Derived.getStatic 存在于 constructor 本身
console.log(Derived.getStatic); // TypeError if not defined on class body
```

### 4. 常见误区与进阶思考
['误区：认为 class 方法定义在类体内就存储在类本身。事实：非静态方法定义在类的 prototype 对象上，每个实例共享同一份函数引用，导致多例状态下 if (a.method === b.method) 为 true。误用闭包捕获实例状态时需特别注意 context 丢失问题。', '误区：混淆 instanceof 运算符的工作机制。instanceof 检查的是对象的原型链上是否包含构造函数的 prototype 属性，而非比较构造函数引用。修改了 Person.prototype = {...} 会导致旧实例的 instanceof Person 返回 false，因为它们的 [[Prototype]] 仍指向旧的原型对象。']
