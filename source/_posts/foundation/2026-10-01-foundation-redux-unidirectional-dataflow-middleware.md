---
title: "每日基础技术总结 · 2026-10-01 · Redux 单向数据流与中间件"
date: 2026-10-01 07:04:22
categories: [技术分享]
tags: ["技术分享", "前端框架层（Vue / React / 工程化）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-01 · Redux 单向数据流与中间件

## 📚 今日主题

> **Redux 单向数据流与中间件**（前端框架层（Vue / React / 工程化））

### 1. 核心概念速览
**定义**：Redux 是一种基于不可变状态树（state tree）与纯函数归约器（reducer）的状态容器实现，其核心机制是单向数据流：action 作为唯一的信息载体被 dispatch，经 reducer 纯函数同步归约出新 state，再由订阅机制通知视图层。

**解决的问题**：在多组件、多异步源共享可变状态时，消除状态更新的非确定性。通过强制状态变迁路径的唯一性与可回溯性，使状态变更成为可记录、可重放、可调试的时序事件流。

**机制本质**：Redux 本质上是事件溯源（Event Sourcing）架构在客户端状态管理上的应用。action 是事件流，state 是对事件流归约后的投影（projection），reducer 是归约函数，middleware 则是事件流上的可组合拦截器（interceptor chain），可在 action 到达 reducer 之前插入任意逻辑（如异步、裁剪、延迟、转发）。

**在计算机体系中的位置**：它属于前端状态协调层，位于 UI 组件树之外，以命令总线（command bus）+ 查询投影（query projection）模式存在。理解它有助于掌握函数式事件流、状态机、CQRS 等后端/分布式概念，是前端工程师向全栈与 AI 系统（如基于状态机的 agent 循环）过渡的思维基石。

**为什么必须掌握**：因为单向数据流与中间件组合是前端领域对『确定性状态变迁与副作用隔离』的最成熟工程实践。专业化工程师需要深刻理解其不可变性、纯函数约定、副作用注入边界，才能在大型应用或自行设计状态架构时做出正确取舍，而非停留在 API 使用层。

### 2. 底层原理剖析
## 核心运行机制（同步基准链路）

```
调用 store.dispatch(action)
  → 中间件链逐个执行（可提前终止或修改 action）
  → 到达根 reducer：reducer(previousState, action) → nextState
  → store 内部用 nextState 替换 currentState
  → 通知所有订阅者（listener）
```

### 关键底层细节

1. **store 内部持有 `currentState` 与 `currentReducer`**。`dispatch` 执行时：
   - `isDispatching` 标志阻止重入（非同步 dispatch 会抛错）。
   - 调用 `currentReducer(currentState, action)` 得到 `nextState`。
   - JS 严格相等 `nextState !== currentState` 决定是否通知订阅（注意：reducer 若返回相同引用则不会触发更新，这是优化的本质依据）。

2. **reducer 纯函数约定**：
   - 输入 `(state, action)` → 输出新 state 对象。
   - 必须保持纯：不产生副作用、不修改 state 参数、不依赖外部随机性。
   - 通过对象展开/不可变数据结构返回新引用。

3. **createStore 内 reducer 的组合机制**：根 reducer 可以是 `combineReducers` 的结果。`combineReducers` 接收一个 map 结构，内部为每个 key 委托其 reducer；若某个子 reducer 返回 `undefined` 则抛错。整个组合过程是同步纯函数归约的递归展开。

## 中间件链的底层构图

Middleware 本质是构造一个包裹原始 dispatch 的级联函数（实质上是函数复合的链式柯里化）：

```
createStore(...) 内部：

middleware = applyMiddleware(m1, m2, m3)

middlewareAPI = {
  getState: store.getState,
  dispatch: (action) => dispatch(action)  // 最终指向原始 dispatch
}

chain = m1(middlewareAPI)(store.dispatch)
       → 返回一个加强版 dispatch，记为 d1
m2(middlewareAPI)(d1) → d2
m3(middlewareAPI)(d2) → d3

最终 store.dispatch = d3
```

每次 dispatch(action) 实际执行链：`m3 内部调用 d2(action) → m2 内部调用 d1(action) → m1 内部调用原始 dispatch(action)`。注意其顺序是**从右向左逐层包裹，从左向右执行**。

## 与前端已有概念的对比

- **与 React 内部更新机制对比**：React 的 setState 依赖于组件实例的私有状态与批处理调度，状态变更路径不透明；Redux 则强制将所有状态变迁集中到 reducer，行为可预测、可测试。React 内部 fiber 调度是异步可中断的，而 Redux 的核心 dispatch 是同步串行的——异步能力只能由中间件注入。
- **与 Java 接口 vs TS 接口的区别类比**：Java 接口是编译时的类型契约，偏重抽象类型与实际实现绑定；TS 接口是结构类型系统（duck typing），只要结构匹配即可。Redux 中间件与 reducer 的关系类似这种类型契约思想——中间件并不强制你实现某个抽象类，而是要求你返回一个符合 `(next) => (action) => any` 结构的函数，这是一种结构化的协议而非身份绑定，这也为它的灵活嵌入提供了可能。
- **与 Vuex 对比**：Vuex 的 mutation 概念将同步变更固化在 store 内部，而 Redux 的 reducer 更接近纯函数投影——Vuex 的 action 可以包含副作用但最终触发 mutation，本质上也是将副作用与同步归约分离，但 Redux 中间件将这一分离提升为可编程的拦截点。

## 执行时序伪代码（精确版）

```
function dispatch(action) {
  if (isDispatching) throw new Error('Reducers may not dispatch actions.')
  isDispatching = true
  try {
    currentState = currentReducer(currentState, action)
  } finally {
    isDispatching = false
  }
  const listeners = currentListeners
  for (let i = 0; i < listeners.length; i++) listeners[i]()  // 订阅通知
  return action
}
```

### 3. 基础代码与实战验证
以下是一个最小化 Redux 核心实现（不含 createStore 内部细节，直接从中间件机制验证单向数据流）：

```javascript
// 1. 基础纯函数 reducer：状态归约器，纯函数无副作用
const initialState = { count: 0 }

function reducer(state = initialState, action) {
  switch (action.type) {
    case 'INCREMENT':
      return { count: state.count + 1 }  // 返回新引用，不可变更新
    default:
      return state  // 未命中 action 时返回原引用，确保不会误触发更新
  }
}

// 2. 极简 createStore：捕获 dispatch 与 subscribe
function createStore(reducer) {
  let state = reducer(undefined, { type: '@@INIT' })
  let listeners = []

  function getState() { return state }
  function dispatch(action) {
    state = reducer(state, action)  // 同步归约
    listeners.forEach(l => l())     // 触发订阅回调
    return action
  }
  function subscribe(fn) {
    listeners.push(fn)
    return function unsubscribe() {
      listeners = listeners.filter(l => l !== fn)
    }
  }
  return { getState, dispatch, subscribe }
}

// 3. 极简中间件实现：logMiddleware 包装 dispatch
// 注意：真实 Redux 中中间件结构为 ({ getState, dispatch }) => next => action => {}
// 此处为教学极简版，直接包裹 dispatch 已验证链式调用与顺序
function logMiddleware(store) {
  return function(next) {
    return function(action) {
      console.log('before:', store.getState())
      const result = next(action)       // 调用下一个中间件或原始 dispatch
      console.log('after :', store.getState())
      return result
    }
  }
}

// 4. 组装：手动构造中间件链（等价于 applyMiddleware 的最终执行链）
const store = createStore(reducer)
const middlewareAPI = {
  getState: store.getState,
  dispatch: (action) => dispatch(action)  // 指向原始 dispatch
}
const enhancedDispatch = logMiddleware(middlewareAPI)(store.dispatch)

// 5. 验证单向数据流与中间件拦截
store.subscribe(() => console.log('订阅者被通知'))
enhancedDispatch({ type: 'INCREMENT' })
// 输出顺序：
// before: { count: 0 }
// 订阅者被通知
// after : { count: 1 }
```

**关键注释**：
- `reducer` 的默认 case 返回原 state 引用，说明如果 action 类型未命中，状态不发生任何变化，也不会触发订阅通知——这是 Redux 优化与可预测性的基础。
- `middlewareAPI.dispatch` 通过箭头函数包装为原始 dispatch，其作用是中间件内部调用的 dispatch 始终指向最底层原始 dispatch，防止中间件链内的后续 dispatch 被再次包裹，造成死循环。
- `logMiddleware` 在三段式柯里化中体现了中间件执行顺序：前段在 `next(action)` 之前执行，后段在 `next(action)` 之后执行，这就是拦截与横切逻辑注入的位置。
- 纯函数 reducer 保证在 `enhancedDispatch` 执行时，state 的变化完全由 dispatch 触发，不依赖任何外部状态——这是单向数据流确定性的根本。

### 4. 常见误区与进阶思考
## 常见认知误区

1. **误以为 middleware 改变了 state 更新本身的性质**
   - 错误认知：middleware 让 Redux 变成了异步的、可以管理异步 action。
   - 事实：middleware 仍然是同步执行的 JS 函数链，它并不能改变 reducer 同步归约的本质。异步能力（如 redux-thunk）只是在 middleware 内部调用底层 dispatch 不再执行下一个中间件后，而是在异步回调中就绪后再调用 dispatch。整个中间件链的包裹逻辑在初始化时已确定，dispatch 本身永远同步。理解这一点，你才能透彻理解为什么 redux-saga 需要 generator 而 thunk 仅仅是让 action 变成函数——它们本质是在为『异步副作用注入一个延迟触发点』，而 Redux 自身并不感知异步。

2. **误以为 dispatch 一个 action 一定会更新 state**
   - 错误认知：action 被 dispatch 后，reducer 会运行且必然产生新状态。
   - 事实：如果 reducer 返回了与当前 state 完全相同的引用（严格相等），store 不会触发订阅通知，因为 Redux 默认使用 `===` 来判断是否需要通知 listeners。这既是不可变数据结构的必然要求，也解释了为什么 reducer 必须返回新对象而不是修改旧对象——修改旧对象会因引用不变而无法触发更新，这是调试困惑（UI 不更新）的深层原因。

## 进阶思考题

假设有两个中间件 A 和 B 通过 `applyMiddleware(A, B)` 注册，A 在 `next(action)` 之前修改了 action 的 `type`，而 B 在 `next(action)` 之前捕获了这个原本的 action 类型。请问最终 reducer 收到的 action 是什么？A 和 B 的执行顺序分别如何？请画出完整的调用链，并解释如果你希望 reducer 收到 A 的改写版本而 B 仍观察原始版本，你需要如何调整中间件的注册顺序与内部结构？这个问题的核心是你是否真正理解了中间件链的包裹顺序、各中间件之间的 action 传递语义，以及 `middlewareAPI.dispatch` 与 `next` 的本质区别——浅层认识很容易给出错误答案，而真正掌握者能直接画出执行时序。
