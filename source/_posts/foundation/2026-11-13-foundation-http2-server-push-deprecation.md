---
title: "每日基础技术总结 · 2026-11-13 · HTTP/2 HTTP2 Push 的原理与浏览器限制：为何现代开发中已不推荐使用"
date: 2026-11-13 07:02:18
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-13 · HTTP/2 HTTP2 Push 的原理与浏览器限制：为何现代开发中已不推荐使用

## 📚 今日主题

> **HTTP/2 HTTP2 Push 的原理与浏览器限制：为何现代开发中已不推荐使用**（前端底层与计算机基础）

### 1. 核心概念速览
HTTP/2 Server Push 是 HTTP/2 协议中由服务器在某个请求流上发送 PUSH_PROMISE，提前承诺并传输另一个响应流的机制。其本质不是双向通道，而是服务器基于已解析依赖关系，替客户端提前发起资源请求。解决的问题是资源发现的串行延迟：浏览器必须先获得并解析 HTML，才能发现 CSS/JS/图片等子资源；HTTP/2 的多路复用只能并行传输已发起的请求，不能消除发现阶段的一次往返。它处在应用层协议、TLS/ALPN、HTTP 缓存、浏览器资源加载器与网络带宽成本的交界处。专业工程师必须掌握它，才能理解为什么现代架构不再用协议猜测代替客户端决策，以及 Early Hints/preload/HTTP/3 的取舍边界。

### 2. 底层原理剖析
1. stream 规则：客户端发起流使用奇数 ID；服务器发起推送流使用偶数 ID；PUSH_PROMISE 必须在原请求流上携带 promise_stream_id。
2. 时序：客户端 GET /index.html；服务器决定 /app.js；服务器向 stream 1 发 PUSH_PROMISE promise_stream=2；再向 stream 2 发 HEADERS+DATA；原始 HTML 可交错发送。
3. 客户端决策：浏览器可接受、丢弃或 RST_STREAM(CANCEL)。判断依据包括 HTTP 缓存、同源或跨域、CSP、资源优先级、连接容量、推送数量限制。
4. 缓存语义：推送资源的缓存键是方法、scheme、authority、path、query；服务器不知道客户端真实缓存，因此可能传输已被缓存或即将被 bust 的资源，造成浪费。
5. 浏览器限制：Chrome 106 移除 HTTP/2 Push；Firefox 136 默认禁用；Safari 支持历史复杂；现代客户端生态已不把它当作可靠优化手段。HTTP/3 规范也移除 HTTP/2 式 Push。
6. 替代：HTTP 103 Early Hints + Link preload/preconnect，由客户端发起 fetch，保留缓存与资源调度权；服务端渲染流式 HTML、合理分包、CDN 和 HTTP/2 多路复用已降低 push 收益。
7. 概念对比：link rel=preload 是客户端声明式预取，浏览器仍可应用缓存、CSP、优先级；Server Push 是传输层提前送字节。WebSocket 是 Upgrade 后持久应用通道，可双向任意消息；Server Push 只在 HTTP/2 响应事务内，不能替代事件推送。

### 3. 基础代码与实战验证
```text
// HTTP/2 Server Push 帧级伪代码；现代浏览器不依赖，仅用于协议验证
client_send:
  send HEADERS { stream_id=1, :method=GET, :path=/index.html } // 客户端发起流：奇数 ID

server_may_push:
  send PUSH_PROMISE { stream_id=1, promise_stream_id=2, headers={ :method=GET, :path=/app.js } } // 在原始请求流上承诺一个服务器发起流：偶数 ID
  send HEADERS { stream_id=2, :status=200, content-type=application/javascript } // 被承诺流的响应头，不是客户端请求的结果
  send DATA { stream_id=2, payload=console.log(1) } // 提前传输子资源字节

server_original_response:
  send HEADERS { stream_id=1, :status=200, content-type=text/html } // 原始 HTML 响应可晚于 PUSH_PROMISE，也可能并行交错
  send DATA { stream_id=1, payload=<script src=/app.js></script> } // 浏览器若接受 push，解析到该 URL 时可命中已接收资源

client_policy:
  if cache_fresh(/app.js): discard stream 2 // 缓存命中或策略不匹配时，推送字节被丢弃
  if not_needed: send RST_STREAM { stream_id=2, error_code=CANCEL } // 客户端可取消未用推送流

verify:
  nghttp -v https://localhost:8443/index.html // 通过 HTTP/2 客户端查看 PUSH_PROMISE/HEADERS/DATA；不证明浏览器实际性能收益
```

### 4. 常见误区与进阶思考
误区一：把 Server Push 当成服务端主动提速的稳定收益。它只在 HTTP/2 有效，且必须被客户端接受、缓存命中、策略允许；现代浏览器已移除或禁用，因此实际生产收益经常为零甚至为负。
误区二：把 HTTP/2 Push 与 WebSocket/SSE 混淆。Push 是单个 HTTP 资源响应的提前传输，不是持续应用事件通道；需要长连接事件推送应使用 SSE/WebSocket/WebTransport 等模型。
思考题：HTTP/3 移除 HTTP/2 式 Push，而转向 103 Early Hints。请从缓存可见性、客户端资源优先级、跨版本与代理兼容、无效字节成本、安全策略控制五个角度，论证为什么提示客户端自己发起请求比服务器替客户端提前发起请求更稳健。
