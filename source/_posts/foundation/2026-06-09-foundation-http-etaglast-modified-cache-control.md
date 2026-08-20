---
title: "每日基础技术总结 · 2026-06-09 · HTTP 缓存协商：ETag/Last-Modified 与 Cache-Control"
date: 2026-06-09 08:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-06-09 · HTTP 缓存协商：ETag/Last-Modified 与 Cache-Control

## 📚 今日主题

> **HTTP 缓存协商：ETag/Last-Modified 与 Cache-Control**（网络基础）

### 1. 核心概念速览
HTTP缓存协商是客户端与服务器之间通过元数据判定资源是否可复用的机制，核心解决网络传输冗余与延迟问题。其本质是将资源版本的『指纹』与『新鲜度』信息随响应下发，由客户端（或中间缓存）在后续请求中携带条件头，服务器据此返回304 Not Modified或200+新资源。ETag是实体标签，强校验器，基于内容哈希或版本号生成，精确到字节级变化；Last-Modified是时间戳，弱校验器，精度到秒且无法感知内容变化（如相同内容不同时间）。Cache-Control是指令集，定义缓存策略（max-age、no-cache、no-store、must-revalidate等），直接决定缓存生命周期与校验行为。三者协作：Cache-Control决定何时需要重新验证，ETag/Last-Modified决定如何验证。该机制处于HTTP应用层，是Web性能优化的基石，与TLS、HTTP/2同属网络基础核心。专业工程师必须掌握，因为它直接影响资源加载速度、服务器负载、用户体验，且是理解CDN、Service Worker、浏览器缓存行为的前提；任何前端性能优化脱离HTTP缓存协商机制，都是空中楼阁。

### 2. 底层原理剖析
底层机制可抽象为两阶段：缓存写入与缓存验证。写入阶段：服务器响应头携带Cache-Control（如max-age=3600）、ETag、Last-Modified，客户端或中间缓存将资源连同这些元数据存入本地。验证阶段：当缓存过期（如max-age到期）或显式要求重新验证（no-cache）时，客户端构造条件请求——若存在Last-Modified则发送If-Modified-Since，若存在ETag则发送If-None-Match。服务器收到条件请求后执行校验：优先比较ETag（强校验，若资源内容未变则匹配），若匹配则返回304空体；若不匹配则返回200完整资源。若仅有Last-Modified，服务器比较时间戳：若资源最后修改时间早于或等于请求中的值则304，否则200。注意：ETag与Last-Modified可同时存在，服务器应优先处理If-None-Match，因为它更精确；若两者都不满足条件则返回200。流程图：请求→本地缓存是否存在？→不存在→200+缓存头→写入缓存；存在→是否新鲜（根据Cache-Control max-age）？→新鲜→200（from cache）→使用；不新鲜→携带If-None-Match/If-Modified-Since发起协商→服务器判断→304→更新缓存元数据（保留实体）→使用；200→替换缓存。对比前端已有概念：ETag之于资源，如同JS中对象的引用标识或React的key，用于识别『同一实体是否变化』，但区别是HTTP缓存协商是网络层的分布式一致性校验，而JS接口是编译期契约。与TS接口的异同：TS接口是静态类型约束，描述形状，不参与运行时；ETag是运行时动态生成的校验值，描述资源状态。但两者共同点在于：都提供了一种『契约』——TS接口约束代码结构，ETag约束资源版本。更贴切的对比是Java的equals/hashCode：ETag类似hashCode，用于快速判断相等；Last-Modified类似equals，但精度低可能误判。Cache-Control则类似于Java中的对象生命周期管理（如WeakReference的回收策略），决定对象何时被GC。核心本质：HTTP缓存协商是『条件请求』的实例化，将『全量拉取』降级为『元数据校验』，利用服务器端资源版本的确定性，换取网络传输的最小化。

### 3. 基础代码与实战验证
```text
以Node.js原生http模块实现一个最小化缓存协商服务器，验证ETag与Last-Modified行为。

const http = require('http');
const crypto = require('crypto');
const fs = require('fs');

const resource = { content: 'Hello HTTP Cache', updatedAt: Date.now() };
// 预计算ETag：基于内容哈希，强校验器。任何字节变化导致ETag变化。
const etag = crypto.createHash('sha256').update(resource.content).digest('hex');

http.createServer((req, res) => {
  if (req.url === '/resource') {
    // 1. 响应头设置缓存策略：max-age=0表示立即过期，必须重新验证；no-cache表示存储但使用前必须验证。
    // 这里用max-age=0 + must-revalidate模拟协商场景。
    res.setHeader('Cache-Control', 'no-cache');
    res.setHeader('ETag', `"${etag}"`);
    res.setHeader('Last-Modified', new Date(resource.updatedAt).toUTCString());

    // 2. 读取客户端条件头。服务器优先处理If-None-Match（ETag强校验）。
    const ifNoneMatch = req.headers['if-none-match'];
    const ifModifiedSince = req.headers['if-modified-since'];

    // 3. 校验逻辑：若If-None-Match与当前ETag完全匹配，返回304空体。
    // 这里演示了ETag优先于Last-Modified的规则。
    if (ifNoneMatch === `"${etag}"`) {
      res.statusCode = 304;
      res.end();
      return;
    }

    // 4. 若没有ETag匹配，再检查Last-Modified。注意：时间精度秒，且不感知内容变化。
    if (ifModifiedSince && new Date(ifModifiedSince).getTime() >= resource.updatedAt) {
      res.statusCode = 304;
      res.end();
      return;
    }

    // 5. 未命中任何条件，返回200和完整资源。
    res.statusCode = 200;
    res.setHeader('Content-Type', 'text/plain');
    res.end(resource.content);
  } else {
    res.statusCode = 404;
    res.end();
  }
}).listen(3000, () => console.log('Server on :3000'));

// 客户端验证：
// 首次请求：GET /resource → 200 + ETag + Last-Modified + Cache-Control: no-cache。
// 第二次请求：带上If-None-Match: "<etag>" → 服务器返回304，响应体为空。
// 实际网络面板可见状态码304，传输体积大幅减少。
// 若修改resource.content（比如改为'Hello HTTP Cache!'），ETag变化，服务器返回200与新内容。
// 注意：max-age=0与no-cache的区别：max-age=0强制立即过期，必须重新验证；no-cache语义为存储但不直接使用，同样必须验证。
// 而no-store才是完全不缓存。
// 伪代码：
// request = { headers: { 'If-None-Match': etag } }
// response = server.handle(request)
// if response.status == 304: use local cache
// else: update local cache with new response
```

### 4. 常见误区与进阶思考
误区1：认为 ETag 一定优于 Last-Modified，因此忽略 Last-Modified。实际上二者是互补关系：ETag 精确但需要服务器计算，Last-Modified 成本极低且足够覆盖大多数时间敏感场景。HTTP 规范允许同时发送，服务器应优先处理 If-None-Match，但客户端可能只支持其一。真实服务器（如 Nginx）默认开启 Last-Modified，ETag 可选，若前端工程师只依赖 ETag 而源站未开启，则缓存协商完全失效，导致每次回源。

误区2：混淆 Cache-Control 的 no-cache 与 no-store。no-cache 并非『不缓存』，而是『缓存但每次使用前必须重新验证』，即强制走协商流程；no-store 才是彻底不缓存。专业工程师常误用 no-cache 以为禁用了缓存，导致性能不升反降。同理，max-age=0 与 no-cache 效果类似，但语义不同：max-age=0 表示资源立即过期，客户端必须重新验证；no-cache 则允许存储但禁止未经验证直接使用。

思考题：假设服务器对某资源返回 Cache-Control: max-age=60, ETag: "v1"，客户端在第 30 秒请求时是否发请求？不发请求，直接使用本地缓存。第 61 秒请求时，客户端会携带 If-None-Match: "v1" 发送条件请求。此时服务器若发现资源未变，返回 304；若资源已变为 "v2"，返回 200 新资源。请进一步思考：如果第 30 秒时用户强制刷新（Ctrl+F5），浏览器通常发送 Cache-Control: no-cache 并携带 If-None-Match，此时服务器应如何处理？若服务器忽略 If-None-Match 直接返回 200，是否违反 HTTP 语义？为什么？
