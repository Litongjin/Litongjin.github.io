---
title: "每日基础技术总结 · 2026-07-27 · Vue 的 ref 与 reactive 的底层差异"
date: 2026-07-27 08:00:00
categories: [技术分享]
tags: ["技术分享", "前端框架层（Vue / React / 工程化）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-07-27 · Vue 的 ref 与 reactive 的底层差异

## 📚 今日主题

> **Vue 的 ref 与 reactive 的底层差异**（前端框架层（Vue / React / 工程化））

### 1. 核心概念速览
ref 与 reactive 是 Vue 3 响应式系统的双核驱动，本质区别在于数据存储层级：ref 基于基本类型封装（Wrapper），内部通过 Proxy 拦截对象属性访问；reactive 直接针对 Object/Array 进行深层 Proxy 代理。ref 解决基本类型无法被 Proxy 直接监测的问题，并保证跨组件引用的同一性；reactive 适用于复杂嵌套数据结构。掌握二者差异是理解 Vue 依赖收集机制、避免内存泄漏及优化渲染性能的基础，体现了从‘值’到‘引用’的响应式抽象层设计哲学。

### 2. 底层原理剖析
1. Proxy 机制限制：ES6 Proxy 仅能拦截对象的自身属性及子属性操作，对 undefined 或原始值（string, number 等）无效。
2. ref 实现原理：内部定义一个中间对象（通常为 { value: val }），利用 `Object.defineProperty` 或 `Proxy` 对该 wrapper 对象的 `value` 属性建立依赖追踪。对外暴露时通过 `.value` 访问实际数据。当传入非普通对象时，内部自动转为 `toRef` 逻辑，确保引用不变。
3. reactive 实现原理：调用 `new Proxy(target, handler)`。Handler 中 intercept get/set/deleteProperty/indexedAccess 等操作。对于嵌套对象，采用递归深度代理（Deep Reactive），即在第一次访问深层属性时动态创建新的 Proxy 实例，而非一次性全量代理，以平衡初始化开销与内存占用。
4. 对比 Java/TS：类似 Java 中 `Integer` (包装类) 与 `int` (基本型) 的关系，但此处是通过语言特性（JS 无指针，靠引用）模拟出的统一响应式契约。Reactive 类似于 Java 的 Bean（直接操作字段），Ref 类似于带有 Getter/Setter 的封装类。

### 3. 基础代码与实战验证
```text
// 验证 Ref 的本质：Wrapper Object
const r = ref(0);
console.log(r.value === 0); // 底层访问的是 r._value 或 r["value"]
console.log(Object.keys(r)); // ["__v_isRef", "value"] 证明其为一个包裹对象

// 验证 Reactive 的深度代理与浅层差异
const o = reactive({ a: 1 });
o.a = 2; // 触发 Proxy handler 中的 set

// 关键误区代码：解构丢失响应性
const { a } = o; 
a = 3; // 此时 a 只是普通数值变量，切断了对原对象的绑定
// 正确做法：使用 toRefs 或保持 .a 引用

// Ref 在模板中的自动解包
// <div>{{ r }}</div> 无需写 {{ r.value }}
// 编译器在编译阶段将 template 中的 r 替换为 r.value
// 但在 JS 逻辑层中必须显式使用 .value
```

### 4. 常见误区与进阶思考
1. 解构响应式对象导致依赖丢失：直接将 `reactive` 对象解构赋值给局部变量（如 `const { name } = state`），会切断 Proxy 连接，后续修改局部变量不会触发视图更新。应使用 `toRefs` 或在操作中始终通过源对象访问。
2. 混用导致的状态不同步：在 `<script setup>` 中，template 会自动解包 ref，但在 TS 严格模式下，若未正确推断类型可能导致 `.value` 访问错误。此外，将 ref 作为 prop 传递时需注意父子组件间的基本类型与引用类型行为差异。
思考题：为什么 Vue 3 选择让 ref 在模板中自动解包，而在 JavaScript 执行上下文强制要求显式访问 .value？这种设计权衡了哪些 DX（开发者体验）与运行时性能的考量？
