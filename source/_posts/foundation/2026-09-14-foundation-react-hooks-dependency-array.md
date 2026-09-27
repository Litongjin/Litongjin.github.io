---
title: "每日基础技术总结 · 2026-09-14 · useState/useEffect 的心智模型与依赖数组"
date: 2026-09-14 08:00:00
categories: [技术分享]
tags: ["技术分享", "前端框架层（Vue / React / 工程化）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-14 · useState/useEffect 的心智模型与依赖数组

## 📚 今日主题

> **useState/useEffect 的心智模型与依赖数组**（前端框架层（Vue / React / 工程化））

### 1. 核心概念速览
useState 是 React 在函数组件中声明可变状态的 Hook，其本质是在 Fiber 节点的 hook 链表上保存一个状态槽，并将更新函数绑定到调度器，使组件在状态变化后重新渲染。useEffect 用于在渲染完成后执行副作用（订阅、请求、DOM 操作等），依赖数组（deps）是决定 effect 是否跳过执行的一组值，通过 Object.is 比较现次渲染的依赖值来判定。
解决的问题：函数组件是纯函数，每次调用没有内部记忆，也无法感知自身生命周期；useState 提供了持久化状态，useEffect 提供了与外部系统同步的时机。
机制：render 阶段按调用顺序将 hook 一次挂载到 Fiber.memorizedState 链表；更新时通过 queue 批量调度；commit 阶段在浏览器绘制前收集 effect，在绘制后异步执行（以及 cleanup）。
在计算机体系中，这属于 UI 框架的状态管理与副作用调度，位于组件模型与渲染管线之间。必须掌握它，因为它是设计 React 应用性能、时序和内存行为的基石，也是学习并发特性（startTransition）的基础。

### 2. 底层原理剖析
Fiber 节点上存在 memorizedState 指向单向链表，每次调用 hook 追加节点。useState 的节点保存 memoizedState 和 queue，setter 会把更新入队，触发调度器和重新渲染；重新渲染时从 queue 中计算出最新值。

useEffect 的节点保存 deps 和 destroy（cleanup 函数）。render 阶段将 effect 挂到 fiber.updateQueue，commit 后在浏览器绘制前同步执行 layoutEffect，在绘制后异步执行 effect，执行前先调用上一次的 destroy。

依赖比较伪代码：

    function areHookInputsEqual(nextDeps, prevDeps) {
      if (prevDeps === null) return false;
      for (let i = 0; i < nextDeps.length; i++) {
        if (!Object.is(nextDeps[i], prevDeps[i])) return true;
      }
      return false;
    }

只有返回 true 才执行 effect。

与 Vue 对比：Vue 的 watchEffect 基于 Proxy 自动追踪依赖；React 依赖数组是手工声明。前者是 “变化即重执行”，后者是 “渲染后比较再决定”。与类组件生命周期对比：useEffect 把 mount 和 update 统一，并且每次 effect 对应一次渲染，闭包捕获了当次渲染的值。

### 3. 基础代码与实战验证
```text
// 极简模拟 React Hooks 的内部机制，不依赖框架

// 全局变量：模拟 Fiber 节点上的 hook 链表，以及当前 hook 索引
let rootFiber = { hooks: [] };
let hookIndex = 0;

// useState：按调用顺序在链表上创建或复用 hook 节点
function useState(initial) {
  const index = hookIndex;
  // 首次渲染创建节点，后续复用已有节点，从而保持状态
  const hook = rootFiber.hooks[index] ?? { value: initial };
  const setValue = (newValue) => {
    // 支持函数式更新，保证拿到的 value 是当前值
    hook.value = typeof newValue === 'function' ? newValue(hook.value) : newValue;
    // 触发重新渲染（真实 React 中会调度更新）
    render();
  };
  rootFiber.hooks[index] = hook;
  hookIndex++;
  return [hook.value, setValue];
}

// useEffect：依赖数组对比后决定是否执行 effect
function useEffect(effect, deps) {
  const index = hookIndex;
  const hook = rootFiber.hooks[index] ?? { deps: undefined, cleanup: undefined };
  // 首次渲染 deps 为 undefined，视为需要执行
  const hasChanged = hook.deps === undefined || deps.some((d, i) => !Object.is(d, hook.deps[i]));
  if (hasChanged) {
    // 执行新 effect 前，先清理上一次的副作用
    if (hook.cleanup) hook.cleanup();
    hook.deps = deps;
    hook.cleanup = effect(); // effect 返回清理函数
  }
  rootFiber.hooks[index] = hook;
  hookIndex++;
}

// 目标组件：验证每次渲染的独立闭包与依赖比较
let api;
function Counter() {
  const [count, setCount] = useState(0);
  useEffect(() => {
    console.log('[effect] 执行，count =', count);
    return () => console.log('[cleanup] 执行，count =', count);
  }, [count]);
  console.log('[render] count =', count);
  api = { increment: () => setCount(count + 1) };
}

// 渲染函数：渲染前重置 hookIndex，使 hook 都能找到自己的链表节点
function render() {
  hookIndex = 0;
  Counter();
}

// 运行：首次渲染，然后模拟一次点击
console.log('--- 首次渲染 ---');
render();
console.log('--- 点击 +1 ---');
api.increment();
// 此时 setCount 内部会调用 render，重新执行 Counter，useEffect 会先执行上一次的 cleanup（count=0），再执行新 effect（count=1）
```

### 4. 常见误区与进阶思考
误区1：把依赖数组当作“响应式监听器”，认为依赖变了 effect 就会立刻执行。实际上 effect 的执行时机是渲染提交之后，依赖数组只用于在渲染后比较是否跳过执行。如果依赖没变但重新渲染，effect 不会执行。
误区2：在 effect 中引用了外部变量但忘记加入依赖数组，导致闭包中捕获的是旧值（陈旧闭包）；反之，将对象/数组/函数字面量直接放入依赖数组，每次渲染都会产生新的引用，导致依赖永远“变化”，可能无限循环或重复执行。
思考题：`useEffect(() => { console.log(count); }, [])` 中打印的 count 为什么永远是初始值？请从 React Fiber 上 hook 节点存储的 deps 为 []，以及 effect 函数本身只定义在首次渲染时的闭包这两个角度解释。
