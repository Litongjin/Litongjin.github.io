---
title: "每日基础技术总结 · 2026-08-26 · 跨域的本质与解决方案"
date: 2026-08-26 06:55:38
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-26 · 跨域的本质与解决方案

## 📚 今日主题

> **跨域的本质与解决方案**（前端底层与计算机基础）

### 1. 核心概念速览
跨域的本质是浏览器同源策略（Same-Origin Policy）对网页脚本的读取权限进行约束。同源指协议、域名、端口三者完全一致；只要任一不同，浏览器便认为跨域。该机制解决的核心问题是：防止一个来源的页面脚本，在用户已登录的情况下，读取另一个来源的敏感数据（例如从银行网站读取余额）。它不是服务端的安全机制，而是浏览器作为用户代理强制实施的客户端安全边界。其运作机制是：浏览器发起跨域 HTTP 请求时自动附带 Origin 头，服务端通过 Access-Control-Allow-Origin 等 CORS 响应头显式声明允许的来源；浏览器只负责根据响应头决定是否将响应体暴露给调用方 JS。在整个计算机体系里，跨域属于 Web 平台安全模型的一部分，与 CSRF、XSS 等威胁模型并列。专业工程师必须掌握它，因为前端与后端的所有接口交互都发生在源边界上；不深入理解 CORS 握手、预检和凭据规则，就无法正确排查线上接口问题，也无法设计安全的开放平台。

### 2. 底层原理剖析
浏览器视角的跨域判定流程（以 fetch 为例）：

1. 计算当前页面 origin = scheme + '://' + host + ':' + port。
2. 请求发起时，分为两类：
   - 简单请求：GET/POST，且 Content-Type 限于 text/plain、multipart/form-data、application/x-www-form-urlencoded，且无自定义请求头。
     流程：直接发送真实请求，请求头自动带 Origin；浏览器检查响应头 Access-Control-Allow-Origin；若该值等于当前 origin（或为 * 且未携带凭据），则把响应体交给 JS，否则抛 CORS 错误。
   - 非简单请求：例如 PUT/DELETE、Content-Type: application/json、携带自定义头 Authorization。
     流程：浏览器先发送 OPTIONS 预检请求，携带 Access-Control-Request-Method 和 Access-Control-Request-Headers；服务端返回 Access-Control-Allow-Methods、Access-Control-Allow-Headers、Access-Control-Max-Age；浏览器校验预检通过后，才发送真实请求，再按简单请求的方式校验真实响应头。
3. 若请求需要携带 Cookie（withCredentials: true）：
   - Access-Control-Allow-Origin 不能为 *，必须精确回显 origin。
   - 服务端必须返回 Access-Control-Allow-Credentials: true。

核心本质：同源策略默认阻止跨源读取，CORS 是服务端通过响应头显式授予的“读取豁免”。请求本身依然会发出、服务端依然会处理，真正被拦截的是浏览器对响应体的读取。

对比前端已有概念：同源策略与 CORS 的关系，类似 Java 接口与 TS 接口的关系——名字相似但层面完全不同。Java 接口是运行时多态契约，TS 接口是编译期结构类型检查；同源策略是浏览器内置的强制安全模型，CORS 是服务端通过响应头对该模型的条件豁免。前者是默认规则，后者是例外放行。理解这种“不同层概念的区分”是排查跨域问题的核心。

### 3. 基础代码与实战验证
```text
// 最小化 CORS 演示：node server.js 启动后，在 http://localhost:3000 页面里 fetch 到 http://localhost:8080。
const http = require('http');

http.createServer((req, res) => {
  // 预检请求：非简单请求（如 PUT + Content-Type: application/json）会先走到这里。
  // 浏览器自动发送 OPTIONS，服务端必须显式回应允许的方法和头。
  if (req.method === 'OPTIONS') {
    res.writeHead(204, {
      // 必须回显来源，不能用 *，因为后续真实请求可能会携带凭据。
      'Access-Control-Allow-Origin': 'http://localhost:3000',
      'Access-Control-Allow-Methods': 'PUT',
      'Access-Control-Allow-Headers': 'Content-Type, Authorization',
      'Access-Control-Max-Age': '600' // 600 秒内同一预检不再重复。
    });
    res.end();
    return;
  }

  // 实际请求：响应头中的 Access-Control-Allow-Origin 是浏览器放行 JS 读取响应的唯一依据。
  if (req.method === 'PUT') {
    res.writeHead(200, {
      'Access-Control-Allow-Origin': 'http://localhost:3000',
      'Content-Type': 'application/json'
    });
    res.end(JSON.stringify({ code: 0, data: 'ok' }));
    return;
  }

  res.writeHead(405).end();
}).listen(8080);

// 浏览器端（从 http://localhost:3000 页面执行）：
// fetch('http://localhost:8080/api', {
//   method: 'PUT',
//   headers: { 'Content-Type': 'application/json', 'Authorization': 'Bearer x' },
//   body: JSON.stringify({ amount: 100 })
// })
// .then(res => res.json())
// .then(console.log);

// 注意：该请求会先触发 OPTIONS 预检；若服务端不处理 OPTIONS，真实 PUT 请求根本不会发出。
// 若要验证简单请求，可将 PUT 改为 POST，Content-Type 改为 text/plain，服务端无需 OPTIONS 分支，但真实响应仍必须带 ACAO 头。
```

### 4. 常见误区与进阶思考
误区 1：认为“跨域是后端问题，前端无解”。实际上同源策略只存在于浏览器环境中，curl、Node.js、App 原生请求都不受限制。前端可以通过开发代理、网关转发或 JSONP 等方式绕过，但生产环境最规范的方案仍是服务端正确返回 CORS 头。

误区 2：把 Access-Control-Allow-Origin 设为 * 就以为万事大吉。当请求携带凭据（withCredentials: true）时，浏览器要求 ACAO 必须是精确回显的 origin，不能是 *，同时服务端还必须返回 Access-Control-Allow-Credentials: true。否则浏览器会直接拦截。

进阶思考题：若 evil.com 向 bank.com 发送一个 POST /transfer，Content-Type 为 text/plain，请求体为 a=1；bank.com 未配置任何 CORS 头。请解释：请求是否到达 bank.com？bank.com 能否完成转账？evil.com 的 JS 能否读取转账响应？为什么？如果 Content-Type 改成 application/json，流程有何不同？这个例子说明了同源策略的主要目标是防止什么？
