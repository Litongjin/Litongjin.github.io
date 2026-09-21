---
title: "每日基础技术总结 · 2024-12-30 · 负载均衡：轮询/一致性哈希/最少连接"
date: 2024-12-30 20:00:00
categories: [技术分享]
tags: ["技术分享", "分布式与架构设计"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-12-30 · 负载均衡：轮询/一致性哈希/最少连接

## 📚 今日主题

> **负载均衡：轮询/一致性哈希/最少连接**（分布式与架构设计）

### 1. 核心概念速览
负载均衡是分布式系统中用于将客户端请求分发到多个后端服务实例的机制，旨在优化资源利用率、最大化吞吐量并最小化响应时间。轮询（Round Robin）是一种无状态的全局分配算法，按顺序循环分配请求；一致性哈希（Consistent Hashing）通过虚拟节点和环形哈希空间实现低负载下的最小数据迁移，常用于缓存或会话保持场景；最少连接（Least Connections）基于当前活跃连接数动态分配，适用于处理时长差异大的异步I/O密集型场景。这三种策略分别解决了静态均匀分布、动态状态感知和小范围故障转移的核心问题，是构建高可用微服务架构的基础设施组件。专业工程师必须掌握它们以理解流量调度背后的数学模型与系统瓶颈。

### 2. 底层原理剖析
1. 轮询：维护一个全局计数器或索引指针。每次请求到来时，选择 (index % N) 对应的服务器，然后 index++。本质是确定性且均匀的几何分布，假设所有节点处理能力相同。
2. 一致性哈希：将 hash 值域抽象为长度为 2^32 的圆环。服务器映射到圆环上。请求也计算 hash 值，顺时针找到第一个遇到的服务器。添加/移除服务器时，仅影响其逆时针方向的下一个服务器上的 key，实现局部变动而非全局重建。前端类比：类似 CSS Grid 中的内容流向，但它是拓扑结构上的连续映射，而非线性数组。
3. 最少连接：实时监控各后端的活跃连接数（active_connections）。新请求分配给 active_connections 最小的节点。若存在平局，通常结合加权轮询。本质是贪心算法在动态负载下的应用，适合长连接或处理耗时不均的场景。
对比前端接口：TS 的 interface 定义契约（类型），JS 的 Interface 在编译期被擦除；负载均衡策略则是运行时的路由决策逻辑。轮询像简单的 switch-case，一致性哈希像复杂的依赖注入容器（根据 Key 定位 Instance），最少连接像 React 的 Fiber 调度器（根据优先级/负荷动态分配渲染任务）。

### 3. 基础代码与实战验证
```text
// 极简一致性哈希演示
const VirtualNodeCount = 150;
const circle = [];
function addToCircle(key, value) {
  for (let i = 0; i < VirtualNodeCount; i++) {
    // 核心机制：对主键进行多次哈希，映射到圆环位置
    const vnodeKey = `${key}-vnode-${i}`;
    const pos = fnv1a_64(vnodeKey) % (Math.pow(2, 32)); 
    circle.push({ pos, realKey: key });
  }
  circle.sort((a, b) => a.pos - b.pos); // 顺时针排序
}
function getServer(key) {
  const serverHash = fnv1a_64(key) % (Math.pow(2, 32));
  // 寻找顺时针第一个大于等于该 hash 值的节点
  for (const node of circle) {
    if (node.pos >= serverHash) return node.realKey;
  }
  // 如果超出最大 hash 值，绕回圆环起点
  return circle[0] ? circle[0].realKey : null;
}
```

### 4. 常见误区与进阶思考
误区一：认为一致性哈希能完全避免数据倾斜。实际上，如果业务数据分布极度不均或虚拟节点数量不足，仍会出现热点节点。误区二：混淆‘最少连接’与‘最低延迟’。最少连接关注的是并发度（Queuing），而非处理速度（Latency），在高并发短连接场景下可能因统计开销过大而失效。
深度思考题：在一个混合了 HTTP 短连接和 WebSocket 长连接的非均质集群中，如何设计一种自适应的负载均衡算法，既能利用最少连接的优势处理长连接，又能保证短连接的极低调度延迟？提示：考虑分离调度器或引入分层的负载度量维度。
