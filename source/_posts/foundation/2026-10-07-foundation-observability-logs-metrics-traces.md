---
title: "每日基础技术总结 · 2026-10-07 · 可观测性：日志/指标/链路追踪三支柱"
date: 2026-10-07 07:02:45
categories: [技术分享]
tags: ["技术分享", "云原生与 DevOps（Docker / K8s）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-07 · 可观测性：日志/指标/链路追踪三支柱

## 📚 今日主题

> **可观测性：日志/指标/链路追踪三支柱**（云原生与 DevOps（Docker / K8s））

### 1. 核心概念速览
可观测性（Observability）是通过系统外部输出推断其内部状态的能力。在分布式系统中，它由三类正交数据支柱构成：日志（Logs）、指标（Metrics）、链路追踪（Traces）。日志记录离散事件的结构化上下文；指标是随时间变化的可聚合数值时间序列；链路追踪描述一次请求在跨服务调用链中的因果传播路径。三者分别解决“发生了什么”“系统整体表现如何”“问题在哪一段传播路径上”的问题。可观测性不是监控的别称，而是对系统黑箱的因果建模能力。对前端工程师而言，它对应从浏览器端埋点（如 Performance API、console.log、Sentry）到后端全链路行为建模的跃迁：前端关注用户侧体验与局部状态，可观测性关注分布式因果拓扑中的状态还原。在云原生与 AI 系统中，它是故障定位、性能归因、SLA 保障与模型服务治理的基础设施层能力，必须掌握。

### 2. 底层原理剖析
三类数据的本质差异在于数据模型与因果粒度：

1. 日志（Logs）：事件流，本质是带时间戳的结构化记录，通常以 trace_id/span_id 关联。数据模型是事件序列，适合高上下文、低聚合场景。底层通常异步写入，经 agent 收集后进入存储索引。
2. 指标（Metrics）：时间序列，本质是维度标签（labels）下的数值聚合。数据模型是 metric_name{label_key=label_value} value timestamp。适合低成本、高频率、可聚合的趋势分析，如 QPS、延迟分位数、错误率。指标通过预聚合降低存储与查询成本。
3. 链路追踪（Traces）：因果图，本质是 Trace 由多个 Span 构成的有向无环图（DAG）。Span 表示一次操作，包含 service、operation、start/end、parent_span_id、tags、events。Trace 表示一次请求的全局因果路径。底层通过上下文传播协议（如 W3C Trace Context）跨进程注入/提取。

与前端已有概念对比：
- 日志 ≈ console.log / 前端埋点，但必须结构化并携带关联 ID；
- 指标 ≈ Web Vitals 聚合统计，但维度标签化并支持时序数据库查询；
- 链路追踪 ≈ PerformanceObserver 的 resource timing，但跨服务、跨进程，构建完整调用树。
关键区别是：前端观测多为单进程内局部观测，可观测性三支柱是跨进程、跨服务的因果建模体系。

### 3. 基础代码与实战验证
```text
// 极简可观测性三支柱验证代码（Node.js，无框架）
const http = require('http');
const { randomUUID } = require('crypto');

// --- 指标：内存与请求计数 ---
const metrics = {
  request_count: 0,
  memory_usage_bytes: () => process.memoryUsage().rss
};

// --- 日志：结构化事件 ---
function log(level, event, ctx = {}) {
  const entry = {
    ts: new Date().toISOString(),
    level,
    event,
    ...ctx
  };
  console.log(JSON.stringify(entry)); // 实际系统中写入文件或采集器
}

// --- 链路追踪：手动构造 trace/span ---
function startSpan(traceId, parentId, name) {
  return {
    trace_id: traceId,
    span_id: randomUUID(),
    parent_span_id: parentId || null,
    name,
    start_time: Date.now()
  };
}

function endSpan(span) {
  span.end_time = Date.now();
  span.duration_ms = span.end_time - span.start_time;
  return span;
}

const server = http.createServer((req, res) => {
  // 从请求头提取 traceparent，若无则新建 trace（模拟 W3C Trace Context）
  const traceparent = req.headers['traceparent'];
  const traceId = traceparent ? traceparent.split('-')[1] : randomUUID();

  // 开始根 span，表示一次 HTTP 请求处理
  const rootSpan = startSpan(traceId, null, 'http_request');

  // 指标：请求计数递增，模拟指标采集点
  metrics.request_count += 1;

  // 日志：记录请求开始，携带 trace_id 与 span_id
  log('info', 'request_start', {
    method: req.method,
    url: req.url,
    trace_id: traceId,
    span_id: rootSpan.span_id
  });

  // 模拟一次下游调用，生成子 span
  const childSpan = startSpan(traceId, rootSpan.span_id, 'db_query');
  setTimeout(() => {
    endSpan(childSpan);

    // 日志：记录子操作完成，保留因果关系
    log('debug', 'span_end', childSpan);

    // 结束根 span
    endSpan(rootSpan);
    log('info', 'request_end', rootSpan);

    // 输出指标快照，模拟 /metrics 接口
    if (req.url === '/metrics') {
      res.setHeader('Content-Type', 'application/json');
      res.end(JSON.stringify({
        request_count: metrics.request_count,
        memory_usage_bytes: metrics.memory_usage_bytes()
      }));
      return;
    }

    res.end('ok');
  }, 20);
});

server.listen(3000, () => {
  log('info', 'server_started', { port: 3000 });
});
```

### 4. 常见误区与进阶思考
误区一：把日志、指标、链路追踪当成同一种监控数据的不同格式。本质差异在于数据模型与因果粒度：日志是事件流，指标是时间序列，追踪是因果图。混用会导致存储成本失控或查询语义错误。
误区二：认为只要打了 trace_id 就具备链路追踪能力。真正的链路追踪依赖上下文传播协议与 span 生命周期管理；若跨进程未正确注入/提取 traceparent，或 span 未正确关联 parent，链路将断裂，无法还原调用路径。

思考题：在微服务架构中，如果某次请求失败，但日志中只有错误信息、指标中只有错误率上升、链路中某 span 缺失，你如何判断是服务异常、网络超时还是上下文传播失败？请基于三支柱的数据模型与因果关系说明排查路径。
