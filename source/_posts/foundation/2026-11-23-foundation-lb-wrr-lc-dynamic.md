---
title: "每日基础技术总结 · 2026-11-23 · 负载均衡器的权重调整算法：加权轮询（WRR）与最小连接数（LC）的动态选择"
date: 2026-11-23 07:01:48
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-23 · 负载均衡器的权重调整算法：加权轮询（WRR）与最小连接数（LC）的动态选择

## 📚 今日主题

> **负载均衡器的权重调整算法：加权轮询（WRR）与最小连接数（LC）的动态选择**（网络基础）

### 1. 核心概念速览
核心概念：负载均衡器的权重调整算法是在后端实例集合上决定下一个请求/连接归属的调度函数。WRR 以配置权重为离散配额，在时间片内按权重比例分配请求；LC 以当前活跃连接数为实时状态，优先选择连接最少的实例。动态选择的本质不是简单二选一，而是根据请求同构性、连接生命周期、后端处理时延方差与健康状态，在静态容量模型与运行时负载模型之间切换或融合。它解决的是：当后端容量异构、请求成本异质、长连接/慢请求存在时，单一静态轮询或纯连接计数会造成局部过载、尾延迟上升与资源利用率下降。体系位置：它位于传输层/应用层反向代理、API 网关、服务网格与 AI 推理网关的数据面调度核心；是理解 QoS、拥塞控制、弹性伸缩、GPU/推理实例并发的基础。前端工程师必须掌握，因为它决定了前端请求最终命中哪个服务实例，直接影响首字节时间、重试策略、连接复用与灰度发布边界。

### 2. 底层原理剖析
运行模型：
1. WRR：每个后端维护 weight 与 currentWeight。每轮为所有节点累加 effectiveWeight，选择 currentWeight 最大者，选中后减去总权重，其余加自身权重，从而把离散权重展开为平滑序列。它只依赖配置，不感知请求执行时长。
2. LC：每个后端维护 activeConnections。请求进入时 +1，请求/连接结束时 -1；选择 active/weight 最小者，使高容量实例可承载更多连接。它感知并发占用，但不感知单请求 CPU、内存或下游依赖成本。
3. 动态选择：数据面持续采集 active、错误率、p99、健康检查、连接增长率；控制面根据规则调整 effectiveWeight 或切换算法。常见策略：请求短且同质时用 WRR；长连接或处理时长方差大时切换到 Weighted LC；也可用复合评分 score = α*(1/weight_norm) + β*(active_norm) + γ*(latency_norm)。
与前端已有概念对比：TS 的 interface 是编译期静态类型契约，运行期不存在；LB 的权重算法是运行期可变状态机与选择函数。更贴近的对照是浏览器事件循环：宏任务队列近似按到达顺序执行，类似静态调度；而调度器根据任务优先级、可中断性与当前占用做决策，类似 LC/复合评分。区别在于：前端调度主要优化交互延迟与单线程公平性，LB 调度优化跨实例吞吐、尾延迟与容量利用率。

### 3. 基础代码与实战验证
```text
type Backend = {
  id: string;
  weight: number;
  currentWeight: number;
  active: number;
};

// 平滑加权轮询：把离散权重展开为均匀序列，避免高权重连续命中造成突发。
function smoothWrr(peers: Backend[]): Backend {
  let total = 0;
  let best: Backend | null = null;
  for (const p of peers) {
    total += p.weight;
    p.currentWeight += p.weight; // 每轮所有候选增加自身有效权重，形成配额累积。
    if (!best || p.currentWeight > best.currentWeight) best = p; // 当前累积最大者获得本轮请求。
  }
  if (!best) throw new Error('empty peers');
  best.currentWeight -= total; // 选中者扣除总权重，使其下一轮不会连续垄断。
  return best;
}

// 加权最小连接：用 active/weight 归一化，容量大的实例允许更多连接。
function weightedLeastConn(peers: Backend[]): Backend {
  let best = peers[0];
  let bestScore = best.active / best.weight;
  for (let i = 1; i < peers.length; i++) {
    const p = peers[i];
    const score = p.active / p.weight; // 连接数除以权重，等价于单位容量负载最低者优先。
    if (score < bestScore) {
      best = p;
      bestScore = score;
    }
  }
  return best;
}

// 动态选择：连接倾斜超过阈值时使用 LC，否则使用 WRR，避免慢连接拖垮静态轮询。
function dynamicPick(peers: Backend[]): Backend {
  const maxActive = Math.max(...peers.map(p => p.active));
  const minActive = Math.min(...peers.map(p => p.active));
  const skewed = maxActive > minActive * 2 && maxActive > 8; // 阈值是防抖与防止小样本抖动。
  return skewed ? weightedLeastConn(peers) : smoothWrr(peers);
}

// 验证：两个慢连接集中在 A 时，动态选择会转向 B；若无倾斜则按权重平滑分发。
const peers: Backend[] = [
  { id: 'A', weight: 5, currentWeight: 0, active: 12 },
  { id: 'B', weight: 3, currentWeight: 0, active: 2 },
];
const next = dynamicPick(peers);
next.active += 1; // 请求进入时增加活跃连接；真实四层/七层代理需在连接关闭或请求结束时 -1。
```

### 4. 常见误区与进阶思考
误区一：把 WRR 权重理解为精确流量比例。权重只决定请求计数分配；如果请求耗时不同，流量比例不等于负载比例，长请求会让某后端连接与队列堆积。误区二：把 LC 的 activeConnections 当作真实资源消耗。连接可能空闲、长轮询、SSE/WebSocket、流式推理都会使连接数与 CPU/显存/下游令牌消耗脱节；必须结合请求权重、处理时长、并发限流和背压。进阶思考：若多个 LB 实例独立维护 active 计数，为什么全局最小连接可能失效？请设计一种基于中心计数、最终一致同步或本地预测补偿的方案，并说明如何在健康检查延迟、指标滞后与切换震荡之间取得稳定性。
