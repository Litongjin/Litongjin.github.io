---
title: "每日基础技术总结 · 2026-08-27 · RESTful API 设计规范"
date: 2026-08-27 06:55:58
categories: [技术分享]
tags: ["技术分享", "AI 开发基础（LLM & Agent）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-27 · RESTful API 设计规范

## 📚 今日主题

> **RESTful API 设计规范**（AI 开发基础（LLM & Agent））

### 1. 核心概念速览
REST（Representational State Transfer）是一种面向资源的分布式系统架构风格，由 Roy Fielding 在博士论文中提出。其本质是将网络中的一切业务实体抽象为“资源”，通过统一接口（HTTP 方法：GET/POST/PUT/PATCH/DELETE）对资源进行无状态操作，并通过超媒体（HATEOAS）驱动客户端在资源状态之间迁移。它解决的核心问题是：如何在不可信、异构的分布式环境中，通过标准化语义实现客户端与服务器的解耦、可缓存性和可伸缩性。在计算机体系中，REST 位于应用层协议（HTTP）之上，是一种架构约束而非实现框架；在 AI 开发中，LLM 的 API 调用、Agent 的工具调用往往遵循 REST 语义，理解它是构建可靠服务边界的基础。专业工程师必须掌握它，因为它是互联网服务交互的事实标准，且能帮助厘清前后端契约的本质。

### 2. 底层原理剖析
REST 的底层机制基于以下核心约束：
1. 资源标识：每个资源通过 URI 唯一定位，例如 /users/42。URI 表示资源身份，不表示动作。
2. 统一接口：使用 HTTP 方法表达操作语义——GET 获取表示（安全、幂等），POST 创建子资源（非幂等），PUT 整体替换（幂等），PATCH 局部更新（非幂等但可设计为幂等），DELETE 删除（幂等）。
3. 无状态性：服务器不保存客户端会话状态，每个请求必须携带所有上下文信息（如认证、分页参数）。这使得任意服务器节点都能独立处理请求，便于水平扩展。
4. 表述（Representation）：服务器返回的是资源的当前状态的一种编码（JSON/XML/HTML），而非资源本身。客户端通过媒体类型（Content-Type/Accept）协商具体格式。
5. 超媒体驱动（HATEOAS）：响应中应包含链接（如 rel="next"），使客户端能从当前状态发现后续操作，避免客户端硬编码 URL。
与前端概念的对比：Java 接口是编译期的类型契约，定义方法签名；TS 接口是结构化的形状约束，两者都存在于代码静态层面。而 REST API 是运行时的网络契约，约束的是 HTTP 请求/响应的语义组合（方法+URI+状态码+表示）。前端工程师熟悉的 axios 调用本质上只是 HTTP 传输，REST 规范则定义了什么才算“正确的调用”和“有意义的响应”。从思维模型上看，Java/TS 接口解决“如何保证函数调用正确”，REST 解决“如何保证分布式资源操作正确”。

### 3. 基础代码与实战验证
下面用 Node.js 原生 http 模块实现一个极简 REST 资源服务，验证方法+URI+状态码的语义：

```javascript
// 严格依赖 HTTP 语义，不引入任何框架
const http = require('http');
let users = [{ id: 1, name: 'Alice' }];
let nextId = 2;

const send = (res, status, data) => {
  res.writeHead(status, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify(data)); // 序列化资源表示为 JSON
};

const parseBody = (req) => new Promise((resolve) => {
  let body = '';
  req.on('data', (chunk) => body += chunk); // 流式收集请求体
  req.on('end', () => resolve(body ? JSON.parse(body) : {}));
});

const server = http.createServer(async (req, res) => {
  const { pathname } = new URL(req.url, 'http://localhost');
  const match = pathname.match(/^\/users\/(\d+)$/); // 资源 URI 模板
  const id = match && Number(match[1]);
  const body = await parseBody(req);

  if (pathname === '/users' && req.method === 'GET') {
    return send(res, 200, users); // 读取集合资源
  }
  if (pathname === '/users' && req.method === 'POST') {
    const user = { id: nextId++, name: body.name };
    users.push(user);
    return send(res, 201, user); // 创建成功，返回新资源
  }
  if (match && req.method === 'GET') {
    const user = users.find(u => u.id === id);
    return user ? send(res, 200, user) : send(res, 404, { error: 'Not Found' });
  }
  if (match && req.method === 'PUT') {
    const idx = users.findIndex(u => u.id === id);
    if (idx === -1) return send(res, 404, { error: 'Not Found' });
    users[idx] = { id, name: body.name }; // PUT 整体替换
    return send(res, 200, users[idx]);
  }
  if (match && req.method === 'DELETE') {
    users = users.filter(u => u.id !== id);
    return send(res, 204); // 无内容，删除成功
  }
  return send(res, 405, { error: 'Method Not Allowed' }); // 统一接口拒绝非法方法
});

server.listen(3000);
```

关键点：路由判断同时依赖 URI 和 HTTP 方法，这正是 REST 统一接口的底层实现方式；状态码（201/204/404/405）承载语义，客户端据此决定下一步行为，而不需要自定义 error code。

### 4. 常见误区与进阶思考
1. 把 REST 当作 CRUD 的 RPC 包装：常见误区是在 URL 中放动词（如 /getUser、/createUser），或把所有写操作都归为 POST。这破坏了统一接口的语义，使客户端必须依赖服务端的私有规则，丧失了 HTTP 标准方法带来的可缓存性和中间件兼容性。真正的 REST 要求：每个资源只有一套 URI，操作由 HTTP 方法表达。
2. 忽视无状态性：在前端工程中，习惯用 Session 或 Cookie 保存登录态，但这违背了 REST 的无状态约束。服务端一旦持有会话状态，就无法独立处理任意请求，导致水平扩展必须引入会话同步或粘性会话。REST 的做法是每次请求携带完整凭证（如 JWT），服务端不存储客户端状态。

思考题：如果一个 GET 请求返回 200 且响应体包含一个 "next" 链接，客户端应当如何使用它？如果此时该链接返回 404，你能否判断是资源不存在、链接过期，还是客户端错误地假设了资源状态？请结合 REST 的超媒体约束和状态码语义解释。
