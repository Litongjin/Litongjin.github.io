---
title: "每日基础技术总结 · 2024-08-22 · Vue3 响应式原理：Proxy 拦截与依赖收集"
date: 2024-08-22 20:00:00
categories: [技术分享]
tags: ["技术分享", "前端框架层（Vue / React / 工程化）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-08-22 · Vue3 响应式原理：Proxy 拦截与依赖收集

## 📚 今日主题

> **Vue3 响应式原理：Proxy 拦截与依赖收集**（前端框架层（Vue / React / 工程化））

### 1. 核心概念速览
Vue3 响应式系统的核心机制基于 ES6 Proxy 对象，旨在通过拦截对目标对象属性的访问（get）、修改（set）、删除（delete）及遍历等操作，实现数据变动通知视图更新。其本质是建立数据与副作用函数（如渲染函数）之间的依赖映射关系。在计算机体系结构中，它属于前端状态管理的观察者模式变种，解决了虚拟 DOM Diff 开销过大问题，实现了细粒度的性能优化。专业工程师必须掌握它，因为理解 Proxy 的元编程能力及其在非原始数据类型上的局限性，是排查深层 Bug、优化应用性能以及理解现代框架设计哲学的前提。

### 2. 底层原理剖析
Proxy 拦截与依赖收集分为两个阶段：Reactive 转换阶段与 Effect 执行阶段。
1. 触发机制：当创建 Vue 实例时，使用 new Proxy(data, handler) 包裹原始数据。handler 中定义 get trap，当属性被读取时，触发依赖收集；定义 set trap，当属性被修改时，触发更新调度。
2. 依赖收集（Track）：在执行计算属性或组件渲染函数（effect）时，会读取响应式对象的属性。此时，全局有一个当前正在执行的 effect 指针（targetMap）。get trap 获取该 effect 并添加到对应属性的 WeakSet 集合中。这建立了 '数据属性 -> [副作用函数列表]' 的映射。
3. 派发更新（Trigger）：当属性被修改，set trap 获取该属性对应的所有副作用函数并执行。由于 Proxy 可以直接捕获自身属性变化，无需像 Vue2 那样递归 Object.defineProperty，因此性能更高且支持数组索引和长度修改。
对比 TS/JS 接口：TypeScript 接口是编译时的类型契约，不生成运行时代码，仅用于静态检查；Proxy 是运行时的动态拦截器，完全改变对象的行为语义。Proxy 允许我们在不修改源数据的情况下注入逻辑（AOP），这是传统继承或混入无法做到的底层控制。

### 3. 基础代码与实战验证
```text
// 简化版 Proxy 响应式核心逻辑验证
const targetMap = new WeakMap(); // 存储 { obj: Map<key, Set<effect>> }
let activeEffect = null; // 全局当前活跃的副作用函数

function reactive(obj) {
  const handlers = {
    get(target, key, receiver) {
      const res = Reflect.get(target, key, receiver);
      track(target, key); // 1. 依赖收集：将 activeEffect 加入 targetMap
      return isPrimitive(res) ? res : reactive(res); // 递归代理嵌套对象
    },
    set(target, key, value, receiver) {
      const oldVal = target[key];
      if (oldVal !== value) {
        trigger(target, key); // 3. 派发更新：执行 targetMap 中注册的 effect
        return true;
      }
      return false;
    }
  };
  return new Proxy(obj, handlers);
}

function track(target, key) {
  if (!activeEffect) return;
  let depsMap = targetMap.get(target);
  if (!depsMap) { targetMap.set(target, (depsMap = new Map())); }
  let dep = depsMap.get(key);
  if (!dep) { depsMap.set(key, (dep = new Set())); }
  dep.add(activeEffect); // 2. 关联副作用
}

function trigger(target, key) {
  const depsMap = targetMap.get(target);
  if (!depsMap) return;
  const dep = depsMap.get(key);
  if (dep) {
    dep.forEach(effect => effect()); // 4. 执行所有依赖该数据的副作用
  }
}
```

### 4. 常见误区与进阶思考
误区1：认为 Proxy 可以拦截所有情况。实际上，Vue3 无法拦截直接替换整个对象（如 list = newList）而非修改其属性，也无法直接拦截非 ownProperty 的动态属性添加（需配合 defineProperty 或返回新 Proxy），且对 Symbol 键的支持在旧版本实现中曾受限。误区2：混淆 Proxy 拦截时机。Proxy 是在属性访问瞬间拦截，而 Dep 类（如 RxJS Subject）通常是手动 emit 通知，前者是隐式的、同步的、由语言特性驱动，后者是显式的、异步可选的、由业务逻辑驱动。思考题：如果在一个响应式对象的 getter 中抛出异常，或者在 setter 中发生阻塞，会对 Reactivity System 的副作用执行队列产生什么影响？请结合 Event Loop 和微任务队列分析 Vue3 的 nextTick 机制为何能解决这种同步更新导致的视图不同步问题。
