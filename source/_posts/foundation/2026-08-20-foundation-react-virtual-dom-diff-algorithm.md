---
title: "每日基础技术总结 · 2026-08-20 · React 虚拟 DOM 与 Diff 算法（同级比较）"
date: 2026-08-20 08:00:00
categories: [技术分享]
tags: ["技术分享", "前端框架层（Vue / React / 工程化）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-20 · React 虚拟 DOM 与 Diff 算法（同级比较）

## 📚 今日主题

> **React 虚拟 DOM 与 Diff 算法（同级比较）**（前端框架层（Vue / React / 工程化））

### 1. 核心概念速览
虚拟 DOM (Virtual DOM) 是真实 DOM 节点的轻量级 JavaScript 对象映射，旨在通过内存计算减少直接操作浏览器的重排 (Reflow) 与重绘 (Repaint) 开销。Diff 算法是 Virtual DOM 的核心机制，用于比较新旧两个 VDOM Tree 的差异并生成最小化补丁集 (Patch Set)。在计算机体系结构中，它属于应用层抽象优化手段，解决了高频率状态更新下的 UI 渲染性能瓶颈。专业工程师掌握此概念的本质在于理解‘声明式 UI’背后的‘过程式实现’，从而在 React 生态中做出更优的性能决策（如 Key 的选择、组件拆分），而非盲目跟随框架行为。

### 2. 底层原理剖析
1. 序列化对比：将复杂的节点树遍历转换为线性数组操作。
2. 同级比较策略 (Same-Level Comparison)：React Diff 算法假设不同位置的子树不会被移动，相同类型的组件产生相同的树结构，不同类型组件产生不同树结构。因此，它只在同一层级进行递归对比，忽略跨层级节点移动的可能性，从而将 O(n^3) 的复杂度降低至 O(n)。
3. 类型判断与属性差异：若节点类型变化，则销毁整个子树；若类型不变，仅比对属性 (Props/Attributes) 的变化，保留现有 DOM 实例以维持焦点状态。
4. Key 的作用：在列表比对中，Key 作为稳定且唯一的标识符 (Stable ID)，替代了基于索引的位置推断，确保元素移动时的最小化重建。
对比前端已有概念：如同 Java 的接口定义行为契约，TS 的 Interface 定义数据结构契约；VDOM 则是 UI 状态的中间表示契约，Diff 是执行该契约的差分引擎。

### 3. 基础代码与实战验证
```text
// 极简伪代码：展示同级 Diff 的核心逻辑
function diff(oldTree, newTree) {
  let patches = {}; // 存储差异补丁
  let index = 0;
  
  // DFS 遍历两棵树，同一下标代表同级节点
  _dfs(oldTree, newTree, index, patches);
  return patches;
}

function _dfs(oldNode, newNode, index, patches) {
  // 1. 记录当前差异类型：无变化、替换、属性变更
  if (!newNode) { /* 删除逻辑 */ }
  else if (oldNode.type !== newNode.type) { patches[index] = { type: REPLACE, node: newNode }; }
  else if (hasPropsChanged(oldNode.props, newNode.props)) { patches[index] = { type: PROPS, props: newNode.props }; }

  // 2. 递归处理子节点 (严格限定在同级 children 数组内)
  if (newNode.children && newNode.children.length > 0) {
    let oldChildren = oldNode.children || [];
    let newChildren = newNode.children;
    
    for (let i = 0; i < Math.max(oldChildren.length, newChildren.length); i++) {
      _dfs(oldChildren[i], newChildren[i], ++index, patches);
    }
  }
}
```

### 4. 常见误区与进阶思考
误区 1：认为 Virtual DOM 一定比直接操作 DOM 快。事实上，VDOM 引入了额外的 JS 对象创建与比对开销，在小规模、低频更新的场景中，直接 DOM 操作可能更具性能优势。其核心价值在于开发效率与可预测性，而非绝对的运行时性能碾压。
误区 2：忽视 Key 的重要性或滥用索引作为 Key。当列表发生排序或删除操作时，使用 Index 作为 Key 会导致不必要的组件卸载与重新挂载，破坏组件状态（如输入框焦点）。只有数据具有唯一自然键（如 UUID）时才应使用 Key，否则应尽量避免对可变数据进行大规模重排。
深度思考题：如果 React 允许任意层的节点移动（打破同级比较假设），Diff 算法的复杂度将如何演变？这种变化对浏览器主线程 (Main Thread) 的事件循环会产生什么具体影响？
