---
title: "每日基础技术总结 · 2026-09-28 · useState/useEffect 的心智模型与依赖数组"
date: 2026-09-28 07:04:39
categories: [技术分享]
tags: ["技术分享", "前端框架层（Vue / React / 工程化）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-28 · useState/useEffect 的心智模型与依赖数组

## 📚 今日主题

> **useState/useEffect 的心智模型与依赖数组**（前端框架层（Vue / React / 工程化））

### 1. 核心概念速览
## 1. 核心概念速览

useState 与 useEffect 是 React 函数组件的两个协议性 Hook：前者提供组件实例上的持久化状态槽，后者声明渲染提交后的副作用任务。useState 的机制不是给普通变量赋值，而是把更新动作追加到一条更新队列中，等待下一次渲染时合并计算；useEffect 的机制也不是响应式监听，而是在渲染提交后比较显式声明的依赖数组，再决定是否重放副作用。两者共同解决了函数组件“每次渲染重新执行函数”与“状态需要跨渲染保存、副作用需要与渲染结果同步”的矛盾。在计算机体系中，它们属于运行时状态管理与副作用调度问题；只有理解这一层，才能真正理解 React 并发渲染、bailout、StrictMode 双调用等进阶现象，而不是停留在 API 记忆层面。

### 2. 底层原理剖析
## 2. 底层原理剖析

### 2.1 Hook 存储结构

每个正在渲染的 Fiber 节点上都有一条 hook 链表；组件函数每次执行时，React 按调用顺序创建或复用链表节点。useState 节点保存 baseState、queue、memoizedState；useEffect 节点保存 create、destroy、deps、next。正因为识别的是“调用顺序/位置”，所以 Hook 不能写在条件、循环或嵌套函数里。

### 2.2 useState 的更新流程

首次渲染时，useState(initial) 创建节点，memoizedState = initial。调用 setState(action) 时，action 被封装成 update 对象追加到该 hook 的 queue 尾部，并调度一次渲染。下一次 render 组件时，React 从头遍历 queue，基于 baseState 结算：若 action 是函数则 newState = action(baseState)，否则 newState = action；结算完成后 memoizedState = newState，queue 被清空。函数式更新能拿到最新 baseState，普通写法只能从当前渲染闭包中读取旧快照。

### 2.3 useEffect 的调度时机

render 阶段只收集 effect；commit 阶段完成并完成浏览器绘制后，React 异步执行 passive effects。React 用 Object.is 逐个比较当前依赖数组与上次依赖数组；只要有一个元素不同，或首次挂载，就依次执行“上一次的清除函数 -> 本次副作用函数 -> 保存新的清除函数”。依赖数组为 [] 时，只有挂载/卸载时执行；省略依赖数组时，每次渲染后都执行。

### 2.4 与前端已有概念的对比

- 和 Vue 的 watch/effect 对比：Vue 依赖响应式系统自动收集依赖；React useEffect 不做依赖追踪，依赖数组是开发者的显式契约。前者是运行时可观测，后者是渲染快照的比对。
- 和事件监听对比：useEffect 不是订阅一个数据源，而是“每轮渲染后核对快照差异再决定是否执行”；数据变化到副作用之间至少隔了一次完整 render。
- 和 class 生命周期对比：useEffect 虽然常被视为 componentDidMount、componentDidUpdate、componentWillUnmount 的合体，但它的控制粒度是依赖数组，不是生命周期阶段；例如 [a] 依赖的 effect 会在 b 更新时跳过，这是生命周期做不到的。
- 这就像 Java 的 interface 与 TypeScript 的 interface：同名且都描述某种契约，但底层的类型检查时机和运行时机制完全不同；Hook 也一样，表面相似的 API 背后是各自独立的渲染模型。

### 2.5 内部运行示意

    // React 内部逻辑示意（非真实源码）
    function useStateWrapper(initialArg) {
      const hook = currentHook;
      hook.memoizedState = hook.queue.pending
        ? processUpdateQueue(hook.queue, hook.baseState)
        : initialArg;
      return [hook.memoizedState, dispatch];
    }

    function useEffectWrapper(create, deps) {
      const hook = currentHook;
      const prevDeps = hook.deps;
      if (!prevDeps || !areDepsEqual(prevDeps, deps)) {
        pendingPassiveEffects.push({ create });
      }
      hook.deps = deps;
    }

    function flushPassiveEffects() {
      for (const effect of pendingPassiveEffects) {
        if (effect.destroy) effect.destroy();
        effect.destroy = effect.create();
      }
    }

    function areDepsEqual(prevDeps, nextDeps) {
      return prevDeps.length === nextDeps.length &&
        prevDeps.every((dep, i) => Object.is(dep, nextDeps[i]));
    }

### 3. 基础代码与实战验证
```text
## 3. 基础代码与实战验证

以下是一个最小 React 组件，只依赖 React 的 hooks 运行时，不引入任何业务框架。

    import React, { useState, useEffect } from 'react';

    function Counter() {
      const [count, setCount] = useState(0);
      const [step, setStep] = useState(1);

      // 依赖数组 [count]：commit 后 React 用 Object.is 比较本次 count 和上次 count。
      // 相等则跳过，不相等则先执行上一次的清理函数，再执行本次副作用。
      useEffect(() => {
        // 这里的 count 是本次渲染闭包捕获的快照，不是最新内存值。
        document.title = `count: ${count}`;

        // 清理函数在下次执行本 effect 之前以及组件卸载时调用。
        return () => {
          console.log('cleanup previous count:', count);
        };
      }, [count]);

      // 空依赖数组：只有挂载阶段执行一次；后续任何 setState 都不会重放。
      useEffect(() => {
        console.log('mounted');
      }, []);

      // 点击按钮不会立即修改状态，而是把更新函数 c => c + step 入队。
      // setCount 调度下一次渲染；step 是本次渲染闭包中的快照。
      return React.createElement(
        'button',
        { onClick: () => setCount(c => c + step) },
        `count: ${count}`
      );
    }

验证点：

1. 首次渲染：两个 effect 的 create 都会执行，mounted 打印一次，title 被设置。
2. 点击一次按钮：React 重新执行 Counter；第一个 effect 的依赖从 0 变成 1，先输出 cleanup previous count: 0，再设置 title；第二个 effect 因依赖 [] 与上次逐项 Object.is 完全相等，被跳过。
3. 多次点击后，cleanup 里打印的 count 永远是上一次渲染的快照。
```

### 4. 常见误区与进阶思考
## 4. 常见误区与进阶思考

### 误区一：把依赖数组当成自动依赖收集

useEffect 不追踪函数体中使用的外部变量，依赖数组只是一个“显式声明的快照比较清单”。例如：

    useEffect(() => setCount(count + 1), []);

这里的 effect 闭包捕获的是首次渲染的 count=0，而 [] 与 [] 经 Object.is 比较永远相等，所以这个 effect 只会执行一次；它不会像 watch 一样因为 count 变化而再次执行。正确写法要么把 count 加入依赖数组，要么改用函数式更新 setCount(c => c + 1)——但即使改成函数式更新，依赖数组为空时 effect 仍只执行一次，这说明数组控制的是“副作用重放频率”，不是“数据来源”。

### 误区二：把 setState 当成同步赋值，把 useEffect 当成生命周期语法糖

setState 不会同步更新组件实例中的某个字段；它只是向 hook 队列追加 update 并调度渲染。同一个事件循环内连续多次 setState 会被批处理，页面只能看到最终状态；若在调用后立刻读取同一渲染闭包中的变量，得到的仍是旧值。useEffect 也不是把三个生命周期方法合并成一句话，它的执行时机、清理时机由渲染提交和依赖比较驱动，跟 class 组件的生命周期阶段没有一一对应关系。

### 进阶思考题

如果 useEffect(cb, [obj]) 中的 obj 是每次 render 都会新建的对象（例如 { id: props.id }），为什么 effect 每次渲染后都会执行？请从 Object.is 比较引用身份和 hook 保存依赖的方式解释，并说明如何在保持依赖值语义不重放的前提下做出正确优化。
