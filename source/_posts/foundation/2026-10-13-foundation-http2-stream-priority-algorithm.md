---
title: "每日基础技术总结 · 2026-10-13 · HTTP/2多路复用（Multiplexing）中流优先权（Stream Priority）算法原理"
date: 2026-10-13 10:05:50
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-13 · HTTP/2多路复用（Multiplexing）中流优先权（Stream Priority）算法原理

## 📚 今日主题

> **HTTP/2多路复用（Multiplexing）中流优先权（Stream Priority）算法原理**（网络基础）

### 1. 核心概念速览
HTTP/2 流优先级是 RFC 7540 定义的、在单条 HTTP/2 连接内为多路复用并发流分配发送资源的建议性机制。它不改变 HTTP 语义，而是在二进制帧层引入 dependency（31 位流依赖）与 weight（8 位权重，实际值 1-256）两个字段，构造一棵以虚拟流 0 为根的依赖树。同一父节点下的子流按 weight 比例竞争父节点获得的发送预算；exclusive 标志使新流可以抢占为某父节点的唯一直接子节点。它解决的问题是：多路复用消除了 HTTP/1.x 的连接级队头阻塞，但共享 TCP 发送窗口使所有并发流竞争同一队列，若按 FIFO 或帧到达顺序调度，关键渲染路径流可能被大体积响应阻塞，因此需要在协议层提供 QoS 信号。该机制位于 HTTP/2 会话层，本质是树形加权公平队列（Hierarchical Weighted Fair Queueing）的协议化表达，是浏览器 Request.priority、preload 等前端资源优先级最终落到传输层的载体。专业工程师必须掌握，否则无法诊断 HTTP/2 下关键资源仍被低优先级资源拖慢的问题。

### 2. 底层原理剖析
底层原理可拆为树构建与加权调度两部分。构建时，每个流记录 parent_stream_id 和 weight。根为虚拟流 0；未指定依赖的流默认依赖流 0，默认 weight 16。收到 HEADERS 或 PRIORITY 帧时，解析 E、Stream Dependency、Weight。如果 E=1，先把该流插入为指定父节点的唯一直接子节点，再把父节点原来的直接子节点全部改挂到该流下，形成局部抢占；如果 E=0，则将该流加入父节点的直接子节点列表，weight 更新。禁止形成循环依赖，否则流关闭。调度时，从根开始，对每个父节点的所有直接子节点执行加权分配：子流 i 可获得父节点预算的 weight_i / Σ weight_siblings。若父节点自身也发送数据，它作为该分配层的一个消费者参与调度（RFC 未强制父节点自身数据与子节点的进一步分配细节）；若父节点被 flow control 阻塞，其配额可能暂时分配给其他可发送分支。典型实现使用 H-WFQ 或加权轮询，为每个流维护虚拟期望发送时间，每次从根选择最小虚拟时间的可发送流发送一个 DATA 帧。

与前端已有概念的异同：它与浏览器资源优先级（如 link rel=preload、fetch(..., { priority: 'high' })）同属调度信号，但 HTTP/2 流优先级是协议层显式字段，由客户端在 HEADERS/PRIORITY 帧中发送，服务器可忽略；浏览器资源优先级是渲染引擎启发式，在发起请求前决定，并可能映射到 HTTP/2 权重，但还会影响连接池、缓存策略等。与 JS 事件循环的宏任务/微任务不同，宏任务/微任务是单线程内执行队列控制，而 HTTP/2 流优先级是跨端点、分布式、建议性的网络队列控制，没有强制抢占语义。

### 3. 基础代码与实战验证
```text
// ===== 简化模型：输出节点预算分配，不区分父节点自身数据 =====
class StreamNode {
  constructor(id, weight = 16) {
    this.id = id;          // 0 为虚拟根流
    this.weight = weight;  // 实际权重 1-256，默认 16
    this.children = [];    // 直接依赖当前流的子流
  }
}

function attach(parent, child, exclusive = false) {
  if (exclusive) {
    const old = parent.children;
    parent.children = [child];
    child.children.push(...old); // 原直接子流整体降级为新流的子流
  } else {
    parent.children.push(child);
  }
}

function allocate(node, budget) {
  const totalWeight = node.children.reduce((sum, c) => sum + c.weight, 0);
  const result = [];
  for (const child of node.children) {
    const share = budget * child.weight / totalWeight; // 同层加权比例
    result.push({ stream: child.id, share });
    if (child.children.length > 0) {
      result.push(...allocate(child, share)); // 子流分到的预算继续向下分配
    }
  }
  return result;
}

// 构建 root 下 A(12)、B(4)，B 下 C(256)
const root = new StreamNode(0);
const A = new StreamNode('A', 12);
const B = new StreamNode('B', 4);
const C = new StreamNode('C', 256);
attach(root, A);
attach(root, B);
attach(B, C);

// root 预算 100：A 得 75，B 得 25；C 作为 B 的唯一子流全部继承 B 的 25。
console.log(allocate(root, 100));

// ===== Node.js 真实 PRIORITY 帧发送示例 =====
const http2 = require('node:http2');
const client = http2.connect('https://example.com');
const req = client.request({ ':path': '/' });
// 发送 PRIORITY 帧：独占依赖 root(0)，weight=256
req.priority({ exclusive: true, parent: 0, weight: 256 });
```

### 4. 常见误区与进阶思考
常见误区：
1. 把流优先级当成硬性 QoS。RFC 7540 明确规定它是建议性（advisory）机制，服务器可以完全忽略或简化实现，因此客户端设置的 weight/依赖不保证端到端生效。尤其在 CDN、网关或反向代理场景下，PRIORITY 帧可能被剥离或重写。
2. 认为 weight 是绝对优先级或固定带宽。weight 只在同一父节点的直接子节点之间比较，且受父节点自身份额上限约束。一个高 weight 的流如果其父节点在更高层只分到很小预算，该流最终可用带宽仍然很小。此外 exclusive 不等于永久独占，它只是把原有兄弟流降为子流。

深度思考题：
假设 root 下有两个子流 A(weight=200) 和 B(weight=1)；B 下只有一个子流 C(weight=256)。A、C 持续发送数据，B 不发送数据，且无 flow control 限制。root 每秒可发送 1000 个 DATA 帧。请计算 A 和 C 的稳态理论帧率，并说明为什么 C 即使 weight=256 仍远低于 A。如果服务器使用粗粒度加权轮询（每次轮转发送 10 帧而不是 1 帧），短期与稳态结果会如何变化？
