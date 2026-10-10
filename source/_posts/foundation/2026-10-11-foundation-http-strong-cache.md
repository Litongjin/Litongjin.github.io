---
title: "每日基础技术总结 · 2026-10-11 · HTTP 缓存：强缓存"
date: 2026-10-11 07:15:24
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础（扩展）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-11 · HTTP 缓存：强缓存

## 📚 今日主题

> **HTTP 缓存：强缓存**（前端底层与计算机基础（扩展））

### 1. 核心概念速览
HTTP 强缓存是 HTTP/1.1 协议层定义的客户端缓存策略：当响应头 Cache-Control 携带 max-age（或 HTTP/1.0 的 Expires 给出绝对过期时间）时，客户端在新鲜度窗口内可复用本地缓存响应，不再向源服务器发起任何请求。它解决的问题是：在“资源不变性可预先声明”的场景下，消除重复网络往返，降低延迟与出口带宽消耗，并减少源服务器负载。其本质是服务端通过响应元数据向客户端授权一段“不必校验”的时间，用一致性换取性能。强缓存与协商缓存共同构成 HTTP 缓存体系：强缓存决定“是否请求”，协商缓存决定“请求后是否返回 304”。专业工程师必须掌握它，因为它是 Web 性能优化、静态资源部署、CDN 缓存策略以及浏览器加载链路分析的底层基础。

### 2. 底层原理剖析
强缓存命中流程：
1. 客户端请求资源前，先查找本地 HTTP 缓存（内存缓存/磁盘缓存）。
2. 若存在缓存条目，读取其响应头中的新鲜度信息。
3. 计算缓存是否仍处于新鲜期：优先取 Cache-Control: max-age=N（相对时间，从响应到达时间起 N 秒内有效）；若无，再取 Expires: <http-date>（绝对时间，受客户端本地时钟影响）。
4. 若 fresh（当前时间 <= 缓存生成时间 + max-age，或当前时间 <= Expires），则直接返回缓存条目，状态码显示 200，来源为 memory cache 或 disk cache；不发网络请求。
5. 若 stale（过期），则进入协商缓存或直接重新请求；这一步已不属于强缓存。

伪代码：
function requestWithStrongCache(req) {
  entry = cacheStorage.match(req.url)
  if (!entry) return networkFetch(req)
  freshness = entry.headers.cacheControl?.maxAge ?? entry.headers.expires
  if (isFresh(entry, freshness)) {
    return entry.response  // from memory/disk cache
  }
  return conditionalRequestOrNetworkFetch(req, entry)
}

与前端已有概念的异同：
- 强缓存 vs Service Worker Cache API：前者由 HTTP 协议栈与浏览器网络层自动处理，不需要 JavaScript 参与，也无法被页面脚本直接读取或修改；后者是脚本级缓存，需要 Service Worker 拦截 fetch 事件并调用 caches.match/put，可编程控制缓存策略，但运行在协议层之上。
- 强缓存 vs 前端路由/组件缓存（如 Vue KeepAlive、React memo）：前者作用于网络资源层，跨页面、跨会话；后者作用于应用内存态，不涉及网络协议，生命周期与应用实例绑定。

一致性关键：强缓存期间服务端若内容变化，客户端感知不到，因此生产环境通常对强缓存资源搭配“文件名内容哈希”或“版本号”变更 URL，强制客户端重新获取。

### 3. 基础代码与实战验证
```text
// 使用 Node.js 内置 http 模块搭建极简服务器，验证强缓存响应头与命中行为
const http = require('node:http')

const server = http.createServer((req, res) => {
  if (req.url === '/app.js') {
    const body = 'console.log(1)'
    // 关键响应头：Cache-Control: max-age=10
    // 表示从响应到达客户端起 10 秒内，客户端可直接用缓存，不发请求
    res.writeHead(200, {
      'Content-Type': 'application/javascript',
      'Cache-Control': 'max-age=10',
      // 可选的 HTTP/1.0 兼容头，优先级低于 Cache-Control，受客户端本地时钟影响
      'Expires': new Date(Date.now() + 10000).toUTCString()
    })
    res.end(body)
  } else {
    res.writeHead(200, { 'Content-Type': 'text/html' })
    res.end('<script src="/app.js"></script>')
  }
})

server.listen(3000, () => {
  console.log('server running at http://localhost:3000')
})

// 验证命令：
// 第一次请求：curl -i http://localhost:3000/app.js
// 会看到 HTTP/1.1 200 OK 与 Cache-Control: max-age=10
// 10 秒内再次请求同 URL，浏览器不会向服务器发送网络请求；
// 在浏览器 DevTools Network 中该请求显示 200 OK (from memory cache) 或 (from disk cache)
// 可通过 server 端日志观察到：第二次命中时没有日志输出，证明未到达服务器
```

### 4. 常见误区与进阶思考
常见误区：
1. 误以为“强缓存未过期就绝对不发网络请求”。实际上强制刷新（Ctrl+F5 / Cmd+Shift+R）会为请求附加 Cache-Control: no-cache 请求头，跳过强缓存的新鲜度判断，强制进入协商缓存或直接重新请求；地址栏回车、普通刷新、前进后退通常遵循强缓存策略。此外，强缓存过期后也不一定立即重新下载完整资源，可能先进入协商缓存并收到 304。
2. 混淆 max-age=0、no-cache 与 no-store：max-age=0 表示缓存条目立即过期，需协商缓存重新验证；no-cache 表示允许缓存但每次使用前必须向服务器验证（即强制协商缓存）；no-store 表示完全禁止缓存。强缓存仅关注 max-age/Expires 的新鲜窗口。

进阶思考：
若将一个静态 JS 文件设置为 cache-control: max-age=31536000（一年），强缓存期间发布了新版本，旧客户端如何确定何时能拿到新资源？为什么业界普遍采用“文件名内容哈希（如 app.a3f2c1.js）+ 永不修改 URL 强缓存一年”的策略，而不是将缓存时间缩短？请从强缓存无法感知服务端变更、URL 作为缓存键、HTML 入口不缓存或短缓存这一链路解释其底层必然性。
