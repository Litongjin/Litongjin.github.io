---
title: "每日基础技术总结 · 2025-10-15 · 设计模式：单例/工厂/策略/观察者"
date: 2025-10-15 20:00:00
categories: [技术分享]
tags: ["技术分享", "分布式与架构设计"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2025-10-15 · 设计模式：单例/工厂/策略/观察者

## 📚 今日主题

> **设计模式：单例/工厂/策略/观察者**（分布式与架构设计）

### 1. 核心概念速览
1. 单例模式 (Singleton): 确保一个类只有一个实例，并提供全局访问点。本质是控制对象生命周期与状态共享，常用于配置管理、连接池等全局资源协调。
2. 工厂模式 (Factory): 定义创建对象的接口，让子类决定实例化哪一个类。本质是解耦对象创建与使用，将‘是什么’与‘怎么做’分离，支持开闭原则。
3. 策略模式 (Strategy): 定义一系列算法，把它们封装起来，并使它们可相互替换。本质是将变化部分封装，固定部分调用，实现运行时多态替代复杂的条件分支。
4. 观察者模式 (Observer): 定义对象间一对多的依赖关系，当一个对象状态改变时，所有依赖者都收到通知并更新。本质是异步解耦的事件驱动机制，降低耦合度，提升扩展性。
这四个模式并非特定语言特性，而是架构设计中应对复杂度、实现关注点分离（Separation of Concerns）的抽象手段，是构建高内聚低耦合系统的基石。

### 2. 底层原理剖析
单例: 通过私有构造函数防止外部new，提供静态方法或属性获取实例。关键在于线程安全（双重检查锁或语言内置机制如JS的单线程+事件循环保证非阻塞场景下的唯一性）。
工厂: 核心是‘创建型’思维。前端TS中对应的是Type Guards（类型守卫）配合泛型约束，后端Java对应接口与实现类的映射表（Map<Class, Function>）。
策略: 核心是‘行为型’继承/组合。前端类似CSS中的Class切换，但更强调逻辑层；后端Java中是Interface定义行为，具体类实现该接口，上下文(Context)持有Interface引用而非具体类。
观察者: 核心是‘发布-订阅’的简化版。前端React/Vue中Props/State或EventEmitter是其自然延伸。后端需关注内存泄漏问题（移除监听器），以及同步vs异步通知的处理。
前端对比: 前端习惯用组件树传递数据（Prop Drilling）或Context解决状态共享，这往往是隐式的紧耦合。设计模式则是显式地管理这种耦合与依赖，特别是在无框架的原生JS或Node.js后端中，必须手动维护这些关系。

### 3. 基础代码与实战验证
```text
// 1. 单例 (基于ES6 Module的单例特性 - 模块级单例)
// module.mjs
const Singleton = { instance: null };
class AppConfig {
  constructor() {
    if (Singleton.instance) return Singleton.instance;
    this.config = { dbHost: 'localhost' };
    Singleton.instance = this;
  }
}
export default new AppConfig(); // 实例化即触发构造，且仅一次

// 2. 工厂 + 策略 结合示例
// 定义策略接口
const Strategies = {
  add: (a, b) => a + b,
  sub: (a, b) => a - b
};

// 工厂函数：根据运算类型返回策略函数
function createCalculator(operation) {
  const strategy = Strategies[operation];
  if (!strategy) throw new Error('Unsupported operation');
  // 返回一个闭包，内部持有策略引用，暴露执行方法
  return {
    execute: (a, b) => strategy(a, b),
    type: operation
  };
}

// 3. 观察者模式基础实现
class EventEmitter {
  constructor() { this.listeners = {}; }
  on(event, fn) {
    if (!this.listeners[event]) this.listeners[event] = [];
    this.listeners[event].push(fn);
    return () => { // 返回取消绑定的回调
      this.listeners[event] = this.listeners[event].filter(f => f !== fn);
    };
  }
  emit(event, ...args) {
    (this.listeners[event] || []).forEach(fn => fn(...args));
  }
}
const emitter = new EventEmitter();
const off = emitter.on('data', (val) => console.log('Got:', val));
emitter.emit('data', 100); // 输出: Got: 100
off(); // 解绑
emitter.emit('data', 200); // 无输出
```

### 4. 常见误区与进阶思考
误区1: 认为单例就是‘变量存为全局’。错误，全局变量容易冲突且无法控制生命周期和初始化顺序。单例的核心在于‘受控的访问入口’和‘延迟初始化/线程安全’。
误区2: 过度设计。在简单脚本或小型应用中强行使用工厂或观察者，反而增加不必要的间接层（Indirection）。只有当对象创建逻辑复杂或有多种实现变体，或者组件间存在长生命周期的松散耦合需求时才引入。
思考题: 在前端React的Hooks（如useState/useEffect）机制下，传统观察者模式是否还有必要？如果我们要实现一个跨多个无关组件的全局状态流，是使用Redux（中间件插件系统，类似高级观察者+发布订阅）还是简单的Context+Reducer？请从内存管理、调试友好度和性能损耗三个维度分析哪种更符合底层架构原则。
