---
title: "每日基础技术总结 · 2026-08-16 · HTTP 缓存策略：Cache-Control 与 ETag"
date: 2026-08-16 08:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-08-16 · HTTP 缓存策略：Cache-Control 与 ETag

## 📚 今日主题

> **HTTP 缓存策略：Cache-Control 与 ETag**（网络基础）

### 1. 核心概念速览
HTTP 缓存策略是通过 HTTP 头部字段（Cache-Control、Expires、ETag、Last-Modified 等）在客户端、代理服务器与源服务器之间建立一套资源副本复用规则。其本质是分布式系统中以时间/指纹为维度的状态同步优化，通过减少重复网络传输来降低延迟与带宽成本。机制分为强缓存（命中后直接使用本地副本，不发起请求）与协商缓存（携带验证条件请求源服务器，304 响应表示可复用）。在整个体系中，它位于应用层 HTTP 协议内部，与 URI、方法、状态码共同构成 HTTP 语义；专业工程师掌握它能精确控制应用性能边界，避免因缓存误判导致的数据陈旧或请求风暴。

### 2. 底层原理剖析
底层运行机制可拆解为两套独立且协作的策略。强缓存：响应头 Cache-Control 的 max-age 或 Expires 指定资源可复用时长，客户端在有效期内直接使用本地副本，不发请求；本质是时间戳过期判断，状态在客户端本地演化。协商缓存：响应头 ETag 提供资源实体的唯一指纹（通常为 hash 或版本号），客户端随后请求时携带 If-None-Match: <etag>，服务器将当前资源指纹与请求值比对；若一致，返回 304 Not Modified（无 body），客户端复用本地副本；若不一致，返回 200 + 新资源。ETag 相比 Last-Modified 更精确，能感知秒级内修改。两者关系：Cache-Control 决定请求是否发出，ETag 决定请求发出后能否免于传输 body。这与前端中的接口契约对比：Cache-Control 更像 Java 接口——编译期（响应阶段）声明规则，强约束、明确失效；ETag 更像 TypeScript 接口——结构类型，运行时（请求验证阶段）通过值匹配决定兼容性。二者并非互斥，而是同一协议的不同层次。请求流程如下：1. 检查本地缓存元数据，若 max-age 未过期且无 no-cache 指令，则直接返回缓存；2. 若过期或存在 no-cache，构造条件请求，附上 If-None-Match（若上次响应有 ETag）；3. 服务器比对指纹，返回 304 或 200；4. 304 时更新缓存元数据并复用 body，200 时替换缓存。

### 3. 基础代码与实战验证
```text
const http = require('http');

const etag = 'v1-abc123'; // 资源指纹

const server = http.createServer((req, res) => {
  if (req.url === '/resource') {
    // 强缓存：60秒内直接使用本地副本，不发起请求
    res.setHeader('Cache-Control', 'public, max-age=60');
    // 协商缓存：为资源提供指纹，供条件请求使用
    res.setHeader('ETag', etag);

    // 客户端如果携带了 If-None-Match，且与当前指纹一致
    if (req.headers['if-none-match'] === etag) {
      res.statusCode = 304; // 无 body，通知复用本地副本
      res.end();
      return;
    }

    res.statusCode = 200;
    res.end('Hello HTTP Cache!');
  } else {
    res.statusCode = 404;
    res.end();
  }
});

server.listen(3000);

验证：node cache.js 启动后，第一次 curl -v http://localhost:3000/resource 返回200及Cache-Control/ETag头。第二次 curl -v -H 'If-None-Match: v1-abc123' http://localhost:3000/resource 返回304。注意：强缓存命中时客户端不会发请求，需通过浏览器或代理验证。
```

### 4. 常见误区与进阶思考
常见误区1：认为 Cache-Control 的 max-age 是绝对安全的时间窗口。实际上，用户手动刷新（F5）或浏览器缓存容量不足时，可能绕过强缓存发起条件请求；同时，CDN 和代理服务器可能忽略 private 指令，导致私有资源被公共缓存。误区2：混淆 no-cache 与 no-store。no-cache 不是不缓存，而是每次使用前必须验证（强制协商缓存）；no-store 才是禁止缓存。思考题：若一个资源响应头为 Cache-Control: no-cache 和 ETag: "abc"，浏览器首次请求后，第二次请求是否会携带 If-None-Match: "abc"？若服务器返回 304，浏览器会重新下载 body 吗？请从缓存生命周期与验证流程的角度解释。
