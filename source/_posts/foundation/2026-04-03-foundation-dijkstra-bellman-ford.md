---
title: "每日基础技术总结 · 2026-04-03 · 图最短路：Dijkstra 与 Bellman-Ford"
date: 2026-04-03 20:00:00
categories: [技术分享]
tags: ["技术分享", "算法与数据结构（面试）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-04-03 · 图最短路：Dijkstra 与 Bellman-Ford

## 📚 今日主题

> **图最短路：Dijkstra 与 Bellman-Ford**（算法与数据结构（面试））

### 1. 核心概念速览
Dijkstra 与 Bellman-Ford 均用于求解加权有向图的单源最短路径问题，但基于不同的松弛策略与假设前提。Dijkstra 基于贪心策略，要求边权非负，通过优先队列高效确定当前距离源点最近且已确定最短距离的节点进行松弛，时间复杂度 O((V+E)logV)；Bellman-Ford 基于动态规划思想，允许负权边，通过对所有边进行 V-1 轮全量松弛以收敛到最优解，并能检测负权环，时间复杂度 O(VE)。在 AI 体系中，它们是强化学习值迭代、搜索算法（如 A*）及路由协议的基础；专业工程师需掌握以处理状态空间搜索及理解数值优化中的收敛条件。

### 2. 底层原理剖析
核心机制均为‘松弛’（Relaxation）：若 dist[u] + weight(u,v) < dist[v]，则更新 dist[v]。

Dijkstra 逻辑：
1. 维护一个最小堆（Priority Queue），存储 (distance, node)。
2. 初始时将源点距离置 0 入堆，其余为 Infinity。
3. 弹出堆顶（当前已知最小距离节点 u），标记为已访问（已确定最终最短距离）。
4. 遍历 u 的邻居 v，执行松弛操作。若 v 未访问且距离被更新，则将新状态推入堆。
本质是贪心选择：一旦节点出堆，其最短路径即固定，不再重新计算。

Bellman-Ford 逻辑：
1. 初始化 dist[src]=0，其余为 Infinity。
2. 循环 V-1 次（V 为节点数）：每次遍历图中所有边 (u,v,w)，若 dist[u] != Infinity 且 dist[u]+w < dist[v]，则更新 dist[v]。
3. 第 V 次遍历用于检测负权环：若仍存在可松弛的边，则存在负权环。
本质是 DP：每一轮迭代确保最多经过 k 条边的路径正确，k 从 1 增至 V-1。

对比前端概念：类似 Vue 响应式系统的两种更新策略。Dijkstra 类似于按需更新（Dependent Update），只处理受变更影响的子树（邻居），效率极高但依赖数据不变性（非负权）；Bellman-Ford 类似于全量 Diff 或批量同步，虽然开销大，但能处理复杂的依赖闭环和异常状态（负权环），保证全局一致性。

### 3. 基础代码与实战验证
```text
// Dijkstra 实现 (使用 Min-Heap)
class PriorityQueue { // 需支持 push 和 poll_min }
function dijkstra(graph, startNode) {
    let dist = new Map();
    let visited = new Set();
    let pq = new PriorityQueue();
    
    // 初始化：源点距离为0，其他为Infinity
    for (let node in graph) {
        dist.set(node, Infinity);
    }
    dist.set(startNode, 0);
    pq.push([0, startNode]);
    
    while (!pq.isEmpty()) {
        let [d, u] = pq.poll(); // 取出当前距离最小的节点
        if (visited.has(u)) continue; // 剪枝：已确定的最短路径不再处理
        visited.add(u);
        
        // 遍历邻居并松弛
        for (let neighbor in graph[u]) {
            let weight = graph[u][neighbor];
            let newDist = d + weight;
            if (newDist < dist.get(neighbor)) {
                dist.set(neighbor, newDist); // 更新距离
                pq.push([newDist, neighbor]); // 加入优先队列等待处理
            }
        }
    }
    return dist;
}
```

### 4. 常见误区与进阶思考
误区：认为 Dijkstra 可以处理负权边。实际上，负权边会破坏贪心策略的单调性（即出堆节点的最终距离可能被后续更小的路径更新），导致结果错误。
误区：混淆 Bellman-Ford 的检测阶段。仅运行 V-1 轮只能得到最短路径，不能判断是否存在负权环，必须额外进行第 V 轮遍历检查是否还能松弛。
思考题：如果图中的边权全部为正，但图非常密集（E ≈ V^2），为什么朴素数组实现的 Dijkstra (O(V^2)) 可能比二叉堆实现的 Dijkstra (O(E log V)) 更快？请从常数因子和缓存局部性角度分析。
