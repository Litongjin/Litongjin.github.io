---
title: "每日基础技术总结 · 2026-11-24 · Dijkstra 算法在负权边场景下的失效证明与 Bellman-Ford 对比"
date: 2026-11-24 08:00:00
categories: [技术分享]
tags: ["技术分享", "算法与数据结构"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-24 · Dijkstra 算法在负权边场景下的失效证明与 Bellman-Ford 对比

## 📚 今日主题

> **Dijkstra 算法在负权边场景下的失效证明与 Bellman-Ford 对比**（算法与数据结构）

### 1. 核心概念速览
Dijkstra 算法解决带权有向/无向图的单源最短路径问题，前提是边权非负。其本质是贪心式标签设定：每次从未确定顶点中选取距离估计最小者，将其状态从临时估计提升为最终最短距离，并用它松弛出边。负权边会破坏『已确定顶点不可能再被改进』的不变量，因此算法可能输出错误距离。Bellman-Ford 以动态规划方式按路径边数进行 n-1 轮全局松弛，不依赖顶点出队顺序，可处理负权边并在第 n 轮检测负权环。它是理解最短路、动态规划、网络流、路由协议、编译数据流分析，以及 AI 中图搜索、价值迭代/策略评估的基础：最短路径不是孤立技巧，而是『状态转移 + 松弛算子 + 收敛条件』的范式。

### 2. 底层原理剖析
Dijkstra 的运行机制可抽象为：维护距离估计 dist，用优先队列从未确定集合中选出最小估计顶点 u，将其固定，再对 u 的出边执行松弛 v = min(dist[v], dist[u] + w(u,v))。正确性证明的核心是：若 u 是被固定的下一个顶点，且存在更短路径 P 到达 u，则沿 P 找到第一个未固定顶点 y。由于所有边权非负，P 中到达 y 的前缀长度不超过 P 的总长度，因此 δ(s,y) <= δ(s,u)。又因为 u 是当前最小估计，dist[u] <= dist[y] <= δ(s,y)，从而推出 dist[u] <= δ(s,u)，与假设矛盾。负权边使前缀长度可能大于总长度，上述不等式链断裂。
失效构造：s->a 权 3，s->b 权 4，b->a 权 -2。真实最短距离 dist[a]=2。Dijkstra 先固定 a=3，再处理 b=4；b->a 松弛得到 2，但 a 已固定，不再更新，输出 3。这不是偶发 bug，而是贪心不变量被负权边反例证伪。
复杂度层面，Dijkstra 配合二叉堆为 O(E log V)，Bellman-Ford 为 O(VE)；这是用更高时间复杂度换取对负权边与负环判定的通用性。
前端工程对照：Dijkstra 的『出队即最终态』类似 TypeScript 类型收窄或 React useMemo 的缓存假设：一次推导被当作终值，依赖输入约束稳定且后续不会反向推翻。负权边相当于已提交状态被后续依赖反向改写，终值缓存失效。Bellman-Ford 则像多轮依赖收敛或脏检查：不承认某个中间结果终态，每轮全局扫描所有边，允许旧估计被新证据覆盖，直到达到不动点。它与 Java 接口和 TS 接口差异同属『约束层次不同』：Java 接口强调名义与运行期实现契约，TS 接口强调结构类型与编译期约束；Dijkstra 是强静态约束下的高性能路径，Bellman-Ford 是弱约束下的通用收敛路径。

### 3. 基础代码与实战验证
```text
const edges = [
  ['s', 'a', 3],
  ['s', 'b', 4],
  ['b', 'a', -2]
];

function dijkstra(nodes, edges, src) {
  const dist = Object.create(null);
  const fixed = new Set();
  for (const n of nodes) dist[n] = Infinity;
  dist[src] = 0;
  while (fixed.size < nodes.length) {
    let u = null;
    for (const n of nodes) {
      if (!fixed.has(n) && (u === null || dist[n] < dist[u])) u = n; // 选择当前最小距离估计，等价于优先队列出队
    }
    if (u === null || dist[u] === Infinity) break;
    fixed.add(u); // 贪心不变量：此后认为 dist[u] 已最终确定，不再接受修正
    for (const [from, to, w] of edges) {
      if (from === u && dist[to] > dist[u] + w) {
        dist[to] = dist[u] + w; // 只从已固定节点向外松弛，负权边造成的反向修正会被忽略
      }
    }
  }
  return dist;
}

function bellmanFord(nodes, edges, src) {
  const dist = Object.create(null);
  for (const n of nodes) dist[n] = Infinity;
  dist[src] = 0;
  for (let i = 1; i < nodes.length; i++) { // 第 i 轮松弛后，最多包含 i 条边的最短路径被正确传播
    for (const [from, to, w] of edges) {
      if (dist[from] !== Infinity && dist[to] > dist[from] + w) {
        dist[to] = dist[from] + w; // 全局松弛：允许任何边在当前轮修正旧估计
      }
    }
  }
  for (const [from, to, w] of edges) {
    if (dist[from] !== Infinity && dist[to] > dist[from] + w) {
      throw new Error('negative weight cycle'); // 第 n 轮仍可松弛，说明存在可达负权环，最短路径无下界
    }
  }
  return dist;
}

// dijkstra(['s','a','b'], edges, 's') => { s: 0, a: 3, b: 4 }，错误
// bellmanFord(['s','a','b'], edges, 's') => { s: 0, a: 2, b: 4 }，正确
```

### 4. 常见误区与进阶思考
误区一：把 Dijkstra 在负权边上的失败归结为实现不严谨，试图通过『允许已出队节点重新入队』修补。此时算法已不是 Dijkstra 的贪心标签设定，而变成通用标签修正/SPFA 类算法；若存在可达负权环，距离会无限下降，终止性依赖额外检测。误区二：把负权边与负权环混淆。负权边可能只增加复杂度但仍存在有限最短路径；可达负权环才使最短路径问题无定义（下界为 -∞）。Bellman-Ford 的价值不只是支持负边，更在于用第 n 轮是否仍可松弛来判定负环。
进阶思考：如果允许对每个节点维护『路径边数』限制，Bellman-Ford 为什么恰好在 n-1 轮后收敛？在不含负环的 n 个顶点图中，任何简单最短路径最多有多少条边？如果存在负环，为什么第 n 轮松弛必然还能发生？
