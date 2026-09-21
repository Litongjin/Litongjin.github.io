---
title: "每日基础技术总结 · 2025-05-23 · 广度优先 BFS 与层序遍历"
date: 2025-05-23 20:00:00
categories: [技术分享]
tags: ["技术分享", "算法与数据结构（面试）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2025-05-23 · 广度优先 BFS 与层序遍历

## 📚 今日主题

> **广度优先 BFS 与层序遍历**（算法与数据结构（面试））

### 1. 核心概念速览
BFS（广度优先搜索）是一种基于队列（Queue）数据结构的图/树遍历算法，核心机制是逐层扩展节点。本质：在非加权图中用于寻找最短路径（最少边数），在树结构中对应层序遍历（Level Order Traversal）。在计算机体系中，它是状态空间搜索的基础，广泛应用于路由协议、网络拓扑发现、AI 中的盲目搜索策略及游戏 AI 的状态树生成。

为何必须掌握：前端工程中涉及 DOM 树解析、复杂组件依赖分析、非递归式异步流程控制时，BFS 提供了确定性的层级访问模型；后端与 AI 中，它是处理图结构数据（如社交网络关系、知识图谱）的基准算法，理解其底层 FIFO 特性是优化内存管理和剪枝策略的前提。

### 2. 底层原理剖析
运行逻辑：
1. 初始化：将起始节点入队，并标记为已访问（Visited），防止循环引用和重复处理。
2. 循环迭代：当队列非空时，执行以下原子操作：
   - Dequeue：取出队首元素 v。
   - Process：对 v 进行业务逻辑处理（如数据读取、条件判断）。
   - Enqueue：将 v 的所有未访问邻接节点 u 依次加入队尾，并立即标记 u 为已访问。
3. 终止：队列为空时结束，确保所有连通分量被覆盖。

底层机制差异：
- 与 DFS（深度优先）对比：DFS 基于栈（Stack/LRU）或隐式调用栈，遵循 LIFO 原则，适合探索单一路径尽头；BFS 基于队列（FIFO），遵循‘先来先服务’原则，保证首次到达某节点的路径即为最短路径（无权图）。
- 与前端概念对比：类似 React 虚拟 DOM 的扁平化 Diff 过程，先处理根节点及其直接子集，再进入下一层。区别于 SQL 中的 CTE（递归查询），BFS 显式管理访问状态，不依赖数据库引擎优化，具有更强的可控性。TS 接口定义 BFS 的行为契约，而实现细节完全由队列的数据一致性保障。

### 3. 基础代码与实战验证
```text
/**
 * 树节点定义
 */
function TreeNode(val) {
  this.val = val;
  this.left = null;
  this.right = null;
}

/**
 * 层序遍历核心实现
 * @param {TreeNode} root
 * @return {number[][]} 按层组织的节点值数组
 */
function levelOrder(root) {
  if (!root) return [];

  // 核心数据结构：队列，维持 FIFO 顺序
  const queue = [root];
  const result = [];

  while (queue.length > 0) {
    // 记录当前层的节点数量，因为后续入队的节点属于下一层
    const currentLevelSize = queue.length;
    const currentLevelValues = [];

    // 精确遍历当前层的所有节点，不混入下一层节点
    for (let i = 0; i < currentLevelSize; i++) {
      // Dequeue：移除队首元素
      const node = queue.shift(); 
      currentLevelValues.push(node.val);

      // 将左子节点入队（若存在）
      if (node.left) {
        queue.push(node.left); // Enqueue 并隐式标记已访问（因仅从未访问节点入队）
      }
      // 将右子节点入队（若存在）
      if (node.right) {
        queue.push(node.right);
      }
    }
    result.push(currentLevelValues);
  }
  return result;
}

/* 关键点注释:
 * 1. shift() 操作的时间复杂度为 O(N)，生产环境建议使用双指针模拟队列以降至 O(1)。
 * 2. 'if (node.left) queue.push(...)' 不仅是添加节点，更是通过“只有出队节点才触发子节点入队”的逻辑，确立了层级边界。
 * 3. 此模式消除了递归调用栈的开销，内存占用取决于最大宽度的节点数，而非树的深度。 */
```

### 4. 常见误区与进阶思考
误区 1：混淆队列大小更新时机。
错误做法：在遍历过程中动态获取 queue.length 作为循环条件。正确做法必须在进入内层循环前缓存 `currentLevelSize`，否则会在循环体内因 push 新节点导致长度增加，造成无限循环或层级混乱。

误区 2：忽视visited集合的重要性。
在图结构（Graph）而非树结构中，若无 visited 集合记录已访问节点，BFS 会在环状结构中陷入死循环或因大量重复计算导致性能崩溃。树结构隐含了无环特性，故代码中常省略显式 visited 检查，但这是图遍历的致命前提。

思考题：
假设有一个 N x M 的二维网格地图，每个单元格值为 0（通）或 1（障碍物），起点为左上角，终点为右下角，求从起点到终点的最短路径步数。请推导如何修改上述 BFS 模板以适应二维坐标体系？特别是如何处理 visited 状态以确保空间复杂度最优？
