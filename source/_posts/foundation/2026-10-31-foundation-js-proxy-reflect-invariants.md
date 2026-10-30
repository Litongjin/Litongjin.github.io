---
title: "每日基础技术总结 · 2026-10-31 · JavaScript Proxy 与 Reflect API：元编程基础与不可配置属性的处理"
date: 2026-10-31 07:01:34
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-31 · JavaScript Proxy 与 Reflect API：元编程基础与不可配置属性的处理

## 📚 今日主题

> **JavaScript Proxy 与 Reflect API：元编程基础与不可配置属性的处理**（前端底层与计算机基础）

### 1. 核心概念速览
**定义与本质**
Proxy 是 ECMAScript 的 exotic object 构造器：`new Proxy(target, handler)` 返回一个代理对象，其内部方法（[[Get]]、[[Set]]、[[HasProperty]]、[[DefineOwnProperty]]、[[GetOwnProperty]]、[[Delete]]、[[OwnKeys]]、[[GetPrototypeOf]]、[[SetPrototypeOf]]、[[IsExtensible]]、[[PreventExtensions]]、[[Apply]]、[[Construct]]）在调用时优先分派到 handler 对应 trap；trap 缺省时转发到 target 的内部方法。Reflect 是把这些内部方法暴露为普通函数的命名空间，提供默认转发、布尔返回值和显式 receiver 语义。不可配置属性处理是 Proxy 的核心不变量（invariant）：引擎在 trap 返回后校验结果，禁止代理伪造 target 上不可配置属性的存在性、值、描述符、键集合与可扩展性状态；违反即抛 TypeError。
**解决的问题**
在运行时拦截并定制对象级操作，而不侵入 target 的属性和原型；Reflect 用于在 trap 中安全复用 target 的默认行为并保持 receiver 正确。
**体系位置**
属于 JS 对象模型与运行时语义层，是元编程基础设施；向上支撑响应式系统、Mock/沙箱、ORM/数据映射、权限代理、不可变视图、AI agent 工具链中的动态对象与 schema 拦截。
**为何必须掌握**
不理解内部方法与 invariant，就无法解释代理在冻结对象、数组、内建对象、getter/setter 上的异常；也无法构建不破坏语言语义的运行时抽象。

### 2. 底层原理剖析
**1. 内部方法与 trap 分派**
JS 对象语义由规范内部方法定义。Proxy 对象对每个内部方法执行：读取 handler[trap]；若为 undefined，调用 target 的对应内部方法；否则 Call(trap, handler, [target, ...args])，再做 invariant 校验。handler 的 trap 与 Reflect 方法基本一一对应：get/set/has/defineProperty/getOwnPropertyDescriptor/deleteProperty/ownKeys/getPrototypeOf/setPrototypeOf/isExtensible/preventExtensions/apply/construct。
**2. Reflect 的职责**
Reflect 不是语法糖，而是内部方法的函数式入口：`Reflect.get(target, key, receiver)` 以 receiver 作为 getter 的 this；`Reflect.set(target, key, value, receiver)` 以 receiver 参与 setter/属性定义；`Reflect.defineProperty`、`Reflect.deleteProperty`、`Reflect.preventExtensions` 返回布尔而非抛错；`Reflect.ownKeys` 返回 string 与 symbol 键；`Reflect.apply/construct` 对应函数调用与构造。trap 内写 `Reflect.get(target, key, receiver)` 是保持代理透明性的基线。
**3. 不可配置属性与不变量**
不可配置属性是对象形状的稳定约束：configurable:false 后不能删除、不能改为可配置、不能改变 enumerable；若同时 writable:false 的数据属性，其值不可变；访问器 get/set 不可替换。Proxy 的 trap 结果必须与 target 的真实内部状态兼容：
- get：target 自有不可配置且不可写数据属性 P，trap 结果必须与 targetDesc.value 为 SameValue。
- set：target 自有不可配置且不可写数据属性 P，trap 返回 true 会触发 TypeError；不能改变该属性。
- has：target 有不可配置自有属性 P 时，不能返回 false。
- getOwnPropertyDescriptor：target 有不可配置自有 P 时不能返回 undefined；返回描述符必须兼容 target 的不可配置描述符（configurable 不能变 true，值/访问器/enumerable 按约束一致）。
- defineProperty：不能把不可配置改为可配置，不能改变不可配置不可写数据属性的值，不能替换不可配置访问器，target 不可扩展时不能新增。
- deleteProperty：target 有不可配置自有 P 时，返回 true 会触发 TypeError。
- ownKeys：结果不能重复，必须包含 target 所有不可配置自有键；target 不可扩展时结果必须恰好等于 target 的自有键集合。
- getPrototypeOf/setPrototypeOf：target 不可扩展时不能伪造原型变化。
- isExtensible/preventExtensions：结果必须与 target 的真实可扩展性一致。
这些校验由引擎执行，不是 handler 自行约定；因此代理不能对冻结/密封对象伪造可写视图。
**4. 与既有概念的差异**
TS interface 是编译期结构化类型，擦除后无运行时实体；Java interface 是运行时名义类型契约与方法分派；Proxy 是运行时对象内部方法拦截，不改变类型系统、不克隆 target、不改变对象身份。Object.defineProperty 是单属性描述符操作；Proxy 是对象级操作拦截，覆盖面更广。Proxy 也不是透明包装：私有字段、内建槽（Date/Map/Set）、this 绑定需通过 receiver、bind 或显式方法调用处理。

### 3. 基础代码与实战验证
```text
// 1) Reflect.get 的 receiver 决定 getter 的 this
const base = {
  _v: 1,
  get v() { return this._v; }
};
const p1 = new Proxy(base, {
  get(target, key, receiver) {
    return Reflect.get(target, key, receiver); // getter 的 this 为 receiver，即 p1
  }
});
console.log(p1.v); // 1；p1._v 再次进入 get trap，最终由 Reflect.get 读取 base._v
// 若写成 Reflect.get(target, key)，getter 的 this 是 base，绕过代理的接收者语义

// 2) 不可配置且不可写数据属性的 get invariant
const target = {};
Object.defineProperty(target, 'id', {
  value: 1,
  writable: false,
  enumerable: true,
  configurable: false
});
const badGet = new Proxy(target, {
  get(t, key, receiver) {
    if (key === 'id') return 2; // 违反 invariant：必须返回 target.id 的 SameValue
    return Reflect.get(t, key, receiver);
  }
});
try {
  console.log(badGet.id);
} catch (e) {
  console.log(e.name); // TypeError
}

// 3) set invariant：不可配置且不可写属性不能被 trap 返回 true 伪装写入
const badSet = new Proxy(target, {
  set() { return true; }
});
try {
  badSet.id = 3;
} catch (e) {
  console.log(e.name); // TypeError
}

// 4) ownKeys invariant：不能遗漏不可配置自有键
const t2 = { a: 1 };
Object.defineProperty(t2, 'hidden', {
  value: 2,
  enumerable: false,
  configurable: false
});
const badKeys = new Proxy(t2, {
  ownKeys() { return ['a']; } // 遗漏不可配置自有键 hidden
});
try {
  Reflect.ownKeys(badKeys);
} catch (e) {
  console.log(e.name); // TypeError
}

// 5) 正确基线：逐 trap 使用 Reflect 转发，保持 target 语义与 invariant
const safe = new Proxy(target, {
  get(t, key, receiver) { return Reflect.get(t, key, receiver); },
  set(t, key, value, receiver) { return Reflect.set(t, key, value, receiver); },
  getOwnPropertyDescriptor(t, key) { return Reflect.getOwnPropertyDescriptor(t, key); },
  ownKeys(t) { return Reflect.ownKeys(t); }
});
console.log(safe.id); // 1
```

### 4. 常见误区与进阶思考
**误区 1：Proxy 可以任意伪造 target 的行为，包括不可配置属性。**
不可配置属性受语言不变量保护。对 Object.freeze/Object.seal 后的对象，get 返回不同值、set 返回 true、deleteProperty 返回 true、ownKeys 过滤不可配置键、getOwnPropertyDescriptor 返回不兼容描述符，都会由引擎抛 TypeError。正确做法是默认用 Reflect 转发；需要隐藏或改写数据时，将数据放在独立存储/可配置属性上，代理只对可配置部分做视图，而不是伪造不可配置属性。
**误区 2：Reflect.get(target, key) 与 Reflect.get(target, key, receiver) 等价，且 Proxy 能拦截一切。**
前者让 getter 的 this 指向 target，后者指向 receiver；在代理链和继承中会改变属性查找与写入位置。Proxy 也不能拦截私有字段访问和内建槽操作，Date/Map/Set 等方法需绑定 target 或显式以 target 调用。
**思考题：** 目标对象已被 Object.freeze，且自有属性 id 为不可配置、不可写数据属性。你希望代理对外把 id 读取为加密字符串，同时允许通过代理写入新属性到旁路 Map。哪些 trap 可以实现，哪些必然违反 invariant？如何重构数据与视图以既满足只读/加密展示，又不触发 TypeError？
