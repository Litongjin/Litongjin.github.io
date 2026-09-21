---
title: "每日基础技术总结 · 2025-11-19 · HTTP 缓存：强缓存（Cache-Control/Expires）与协商缓存（ETag/Last-Modified）"
date: 2025-11-19 20:00:00
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础（扩展）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2025-11-19 · HTTP 缓存：强缓存（Cache-Control/Expires）与协商缓存（ETag/Last-Modified）

## 📚 今日主题

> **HTTP 缓存：强缓存（Cache-Control/Expires）与协商缓存（ETag/Last-Modified）**（前端底层与计算机基础（扩展））

### 1. 核心概念速览
HTTP 缓存是浏览器与服务器之间基于资源版本控制的优化机制，旨在减少网络往返（RTT）和带宽消耗。其本质是通过客户端存储资源副本，并在后续请求中验证该副本的有效性来决定是否使用本地缓存或重新获取最新资源。强缓存由 Cache-Control 或 Expires 响应头控制，命中时直接返回 200 OK 且无需与服务器通信；协商缓存由 ETag/Last-Modified 响应头控制，命中时需向服务器发送条件请求（带有 If-None-Match 或 If-Modified-Since），若服务器判定未修改则返回 304 Not Modified，否则返回 200 及新资源。专业工程师必须掌握它以理解分布式系统中一致性、性能与延迟的权衡，这是构建高可用后端服务和 AI 模型推理服务底层数据传输优化的基石。

### 2. 底层原理剖析
机制流程如下：
1. 首次请求：浏览器解析 HTML/CSS/JS 等资源 URL，发起 HTTP GET 请求。服务器响应包含资源 Body 及缓存控制头（如 Cache-Control: max-age=3600, ETag="abc123"）。浏览器将资源写入磁盘或内存缓存数据库。
2. 强缓存判断：当资源再次被引用时，浏览器首先检查本地缓存时间戳。若当前时间 < (最后访问时间 + max-age) 或 < Expires 指定时间，视为强缓存命中。此时跳过网络请求，直接从本地读取资源，状态码标记为 200 (from disk cache / from memory cache)。
3. 协商缓存判断：若强缓存过期，浏览器发起新的 HTTP GET 请求，但请求头会携带上次响应的验证信息：If-None-Match: "abc123" (对应 ETag) 或 If-Modified-Since: Thu, 01 Jan 1970 00:00:00 GMT (对应 Last-Modified)。
4. 服务器校验：服务器接收请求，计算资源的当前版本号并与请求头比对。若一致，返回 304 No Content，无 Body；若不一致，返回 200 OK 及新资源 Body，并更新缓存头。
对比前端 TS 接口与 Java 接口的差异：TS 接口是编译时的静态契约，用于类型检查，不产生运行时开销；Java 接口是运行时多态的实现基础，定义对象行为协议。HTTP 缓存头则是运行时协议的状态约定，ETag/Cache-Control 类似于函数签名中的参数类型约束（严格匹配），而协商过程类似于接口方法调用中的空指针检查与返回值校验，确保数据一致性而非仅结构正确性。

### 3. 基础代码与实战验证
```text
// Node.js Express 示例模拟缓存逻辑
app.get('/api/data', (req, res) => {
  const resource = { content: 'some data', version: 'v1' };
  const etag = JSON.stringify(resource.version); // 生成唯一标识
  const lastModified = new Date().toUTCString(); // 最后修改时间

  // 1. 处理协商缓存请求
  if (req.headers['if-none-match'] === etag) {
    return res.status(304).end(); // 命中协商缓存，返回 304
  }
  if (req.headers['if-modified-since']) {
     // 简化处理，实际需精确比较时间戳
     return res.status(304).end(); 
  }

  // 2. 设置强缓存与协商缓存头
  res.set('Cache-Control', 'public, max-age=3600'); // 强缓存 1 小时
  res.set('ETag', `"${etag}"`);                   // 内容指纹
  res.set('Last-Modified', lastModified);         // 时间戳
  
  // 3. 返回资源
  res.json(resource);
});

// 客户端 fetch 验证步骤
fetch('/api/data')
  .then(res => {
    console.log('Status:', res.status); // 首次 200，二次强缓存前 304 或 200
    if (res.status !== 304) return res.json();
    return null; // 304 时无 body
  });
```

### 4. 常见误区与进阶思考
误区一：认为 Cache-Control: no-cache 表示不缓存，实则它表示强制协商缓存（每次使用前都需向服务器验证），而非禁用缓存；no-store 才是彻底不缓存。误区二：过度依赖 Last-Modified，其在秒级精度下可能因系统时钟漂移或编辑后内容不变导致错误失效；应优先使用 ETag 基于内容的哈希校验，精度更高。思考题：在微服务架构中，如果多个网关节点或 CDN 边缘节点分别持有不同版本的 ETag，客户端从不同路径访问同一资源可能导致频繁的回源请求甚至缓存穿透，如何通过设计统一的版本管理中心或分布式共识算法来解决这一跨节点缓存一致性问题？
