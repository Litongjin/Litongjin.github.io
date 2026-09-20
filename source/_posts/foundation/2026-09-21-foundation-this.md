---
title: "每日基础技术总结 · 2026-09-21 · this 绑定的四条规则与箭头函数例外"
date: 2026-09-21 07:02:57
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础（扩展）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-21 · this 绑定的四条规则与箭头函数例外

## 📚 今日主题

> **this 绑定的四条规则与箭头函数例外**（前端底层与计算机基础（扩展））

### 1. 核心概念速览
this 是 JavaScript 中一个动态绑定的词法环境指针，其本质并非定义在声明时确定，而是在函数执行上下文中运行时由调用栈决定。它解决的核心问题是：如何在非面向对象语法的语言中实现方法复用与上下文隔离。四条规则（默认、隐式、显式、new）构成了传统函数的绑定优先级体系；箭头函数作为例外，不拥有独立的 this 绑定，而是继承外层词法作用域的 this，这使其成为闭包场景下的确定性上下文工具。掌握它是理解 JS 事件循环、原型链方法调用及异步回调中上下文丢失问题的基石。

### 2. 底层原理剖析
1. 默认绑定 (Default Binding): 独立函数调用。在非严格模式下绑定全局对象 (window/global)，严格模式下为 undefined。优先级最低。
2. 隐式绑定 (Implicit Binding): 作为对象方法调用。this 指向调用该方法的对象所有者。若多层点操作符，仅最后一层影响 this。赋值或别名调用会退化为默认绑定。
3. 显式绑定 (Explicit Binding): 通过 call, apply, bind 强制指定 this。bind 返回新函数并预设 this，call/apply 立即执行。
4. new 绑定 (New Binding): 使用 constructor 模式调用。创建新对象，将 this 指向新对象，若构造函数无返回值则返回新对象，若有复杂对象返回值则覆盖 this。

优先级: new > 显式 > 隐式 > 默认。

箭头函数机制: arrowFunc = () => {...} 在定义时捕获当前执行环境的 Lexical Environment 中的 this 引用，后续无论何种方式调用，this 恒定不变。它没有 arguments 对象，需使用 rest 参数替代。

### 3. 基础代码与实战验证
```text
// 验证不同绑定规则及箭头函数的词法继承特性

const obj = {
  id: 'A',
  regular: function() { return this.id; },
  arrow: () => { return this.id; }
};

// 1. 隐式绑定: this -> obj
console.log(obj.regular()); // 'A'

// 2. 默认绑定: 解构后独立调用，非严格模式下 this -> window/global
const standalone = obj.regular;
console.log(standalone()); // undefined (浏览器) 或 GlobalID

// 3. 显式绑定: 强制 this -> 外部对象
const extObj = { id: 'B' };
console.log(obj.regular.call(extObj)); // 'B'

// 4. new 绑定: this -> 新建实例
function Person(id) { this.id = id; }
const p = new Person('C');
console.log(p.id); // 'C'

// 5. 箭头函数例外: 捕获定义时的外层 this
const outer = {
  id: 'Outer',
  getArrow: function() {
    // 此处的 this 指向 outer (隐式绑定)
    const inner = () => {
      // 内部箭头函数捕获外层的 this
      return this.id;
    };
    return inner();
  },
  getRegular: function() {
    const inner = function() {
      // 常规函数此时为默认绑定 (undefined)
      return this ? this.id : 'lost';
    };
    return inner();
  }
};
console.log(outer.getArrow()); // 'Outer' (成功继承)
console.log(outer.getRegular()); // 'lost' (this 丢失)
```

### 4. 常见误区与进阶思考
误区: 认为箭头函数完全没有 this，误以为可以在箭头函数中通过 call/bind 改变其行为。实际上箭头函数有 this，只是不可改变（read-only），因为它是在词法解析阶段确定的。
进阶思考: 在 React 类组件的构造函数中，为什么必须对生命周期方法使用 .bind(this) 或箭头函数？如果直接在 render 中定义普通函数作为事件处理程序绑定到 DOM 元素，会发生什么现象？请从 V8 引擎的闭包实现和事件委托机制角度分析性能与内存泄漏风险。
