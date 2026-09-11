---
title: "每日基础技术总结 · 2026-09-11 · RESTful API 设计规范"
date: 2026-09-11 18:32:46
categories: [技术分享]
tags: ["技术分享", "AI 开发基础（LLM & Agent）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-11 · RESTful API 设计规范

## 📚 今日主题

> **RESTful API 设计规范**（AI 开发基础（LLM & Agent））

### 1. 核心概念速览
RESTful API 是一种基于 HTTP 协议、以资源为中心的分布式系统接口设计风格。其本质是将应用功能抽象为对资源的统一操作，利用 HTTP 标准方法（GET/POST/PUT/DELETE 等）表达语义动作，通过 URI 标识资源，并以超媒体作为状态转移的引擎。它解决的核心问题是客户端与服务端之间松散耦合、无状态、可缓存的数据交互。在整个计算机体系中，REST 是 Web 架构的关键约束，属于现代 API 设计的基础范式。专业工程师必须掌握它，因为它是 HTTP 语义的真正运用，而非简单的 URL 模板约定。

### 2. 底层原理剖析
底层原理建立在 HTTP 协议语法之上。每个资源对应一个 URI，客户端通过 HTTP 方法对资源施加语义操作：GET 幂等获取表示，PUT/DELETE 幂等，POST 非幂等创建或触发。REST 要求无状态：每个请求包含完整上下文，服务端不保存客户端会话状态。表示（Representation）是资源在特定媒体类型下的状态快照，如 JSON/XML。超媒体约束（HATEOAS）使响应中包含可用的后续动作链接，真正实现状态转移。对比前端已有的‘接口’概念：Java/TS 中的 interface 是编译期类型契约，用于约束代码结构；REST 中的接口是运行时的网络端点，通过 URI + 方法 + 媒体类型定义，并通过 HTTP 状态码表达结果。两者处于不同抽象层次：前者是语言类型系统的静态约定，后者是分布式系统通信的协议级约定。

### 3. 基础代码与实战验证
```text
用 Node.js 原生 http 模块实现最小 RESTful API。
代码：
const http = require('http');
const items = [{ id: 1, name: 'a' }];
http.createServer((req, res) => {
  const { method, url } = req;
  if (url === '/items' && method === 'GET') {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify(items)); // 返回资源表示
  } else if (url === '/items' && method === 'POST') {
    let body = '';
    req.on('data', chunk => body += chunk); // 流式接收请求体
    req.on('end', () => {
      const obj = JSON.parse(body);
      obj.id = items.length + 1;
      items.push(obj);
      res.writeHead(201, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify(obj)); // 返回新资源表示，状态码 201 创建成功
    });
  } else {
    res.writeHead(404, { 'Content-Type': 'text/plain' });
    res.end('Not Found'); // 资源或方法不支持时返回 404
  }
}).listen(3000);
该代码直接解析 HTTP 请求的 method 和 url，按 REST 语义分发：GET 幂等，POST 创建。未使用任何框架，展示底层请求-响应机制及状态码用法。
```

### 4. 常见误区与进阶思考
误区1：将 RESTful API 设计等同于 URL 风格，只注意路径用名词复数、用 HTTP 方法，而忽略无状态和超媒体约束。实际上，无状态是协议强制，超媒体才是区分成熟 REST 的关键指标。误区2：把业务操作直接映射为 HTTP 方法，却没有考虑幂等性，例如用 PATCH 做部分更新时不保证重试安全性，导致客户端重试造成数据不一致。思考题：如果客户端通过 GET 请求执行一个删除操作（如 GET /items/1?delete=1），服务端成功删除了资源，但该请求随后被 CDN 或浏览器缓存，另一次 GET 会命中缓存返回已删除的表示——这如何违背 REST 的语义和无状态原则？
