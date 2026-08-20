---
title: "每日基础技术总结 · 2026-06-17 · CORS 预检请求与 Access-Control-Allow-Origin 反射"
date: 2026-06-17 08:00:00
categories: [技术分享]
tags: ["技术分享", "安全基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-06-17 · CORS 预检请求与 Access-Control-Allow-Origin 反射

## 📚 今日主题

> **CORS 预检请求与 Access-Control-Allow-Origin 反射**（安全基础）

### 1. 核心概念速览
CORS（Cross-Origin Resource Sharing）是浏览器基于同源策略的安全边界之上，为跨域资源共享制定的协商协议。预检请求（Preflight Request）是浏览器在发起“非简单请求”前自动发送的OPTIONS请求，用于向服务端询问是否允许实际请求的方法与头。Access-Control-Allow-Origin反射是指服务端将请求头中的Origin字段原样回写至响应头，从而动态声明允许的来源。该机制解决的核心问题是：在不破坏同源策略的前提下，通过服务端显式授权实现跨域访问。其本质是浏览器作为策略执行者，与服务端通过HTTP头完成一次权限协商。在计算机体系中，它属于Web安全模型中的跨域策略层，与CSRF、CSP、SameSite等共同构成浏览器安全防线。专业工程师必须掌握，因为前后端分离架构中跨域场景普遍，错误配置会直接导致安全漏洞（如数据泄露）或功能不可用（如接口被浏览器拦截），且预检请求的细节往往在性能优化与安全审计中成为关键。

### 2. 底层原理剖析
浏览器将跨域请求分为两类：简单请求（Simple Request）和非简单请求。简单请求满足：方法为GET/HEAD/POST，Content-Type为application/x-www-form-urlencoded、multipart/form-data或text/plain，且无自定义请求头。对于简单请求，浏览器直接发送实际请求，但会校验响应中的Access-Control-Allow-Origin是否匹配当前页面Origin，不匹配则拦截。对于非简单请求，浏览器先发送预检请求（OPTIONS），请求头携带Origin、Access-Control-Request-Method（实际请求方法）和Access-Control-Request-Headers（实际请求的自定义头列表）。服务端收到后，需响应Access-Control-Allow-Origin、Access-Control-Allow-Methods、Access-Control-Allow-Headers等。浏览器验证这些响应头是否覆盖实际请求的方法和头，并检查Origin是否匹配，全部通过后才发送真实请求；否则控制台报CORS错误，且实际请求根本不会发出。

预检请求本身不会携带业务数据，其目的仅是“试探”。服务端即使返回错误，也不会影响资源本身，因为预检请求是独立的一次HTTP事务。

Access-Control-Allow-Origin反射是服务端的一种配置方式：读取请求头Origin，直接将其值写入响应头。这与编程语言中的反射（Reflection，程序运行时检查自身结构）有本质区别：CORS反射是数据回显，语言反射是元信息自省。前者作用于HTTP协议层，后者作用于运行时类型系统。类比前端工程师熟悉的Java接口与TS接口：两者名字相似，但一个是运行时的约束（Java接口是编译期类型契约），一个是类型层面的结构约束（TS接口在编译期被擦除），同样，CORS反射与编程反射也仅在名称上有交集，机制与用途完全不同。

### 3. 基础代码与实战验证
```text
const http = require('http');
http.createServer((req, res) => {
  const origin = req.headers.origin; // 读取请求头中的Origin，若无则undefined
  // 预检请求的特征：方法为OPTIONS且带有Access-Control-Request-Method头
  if (req.method === 'OPTIONS' && req.headers['access-control-request-method']) {
    // 反射Origin：将请求方声明的来源原样返回，浏览器会据此判定是否匹配当前页面源
    res.setHeader('Access-Control-Allow-Origin', origin || '*');
    // 声明允许的实际请求方法
    res.setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE');
    // 声明允许的自定义头，需覆盖实际请求中的头
    res.setHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization');
    // 预检结果缓存1小时，减少后续预检次数
    res.setHeader('Access-Control-Max-Age', '3600');
    res.statusCode = 204; // 204表示预检成功
    res.end();
    return;
  }
  // 实际请求（简单请求或预检通过后的真实请求）
  if (req.url === '/api/data') {
    // 同样反射Origin，保证跨域响应可被浏览器接受
    res.setHeader('Access-Control-Allow-Origin', origin || '*');
    res.setHeader('Content-Type', 'application/json');
    res.end(JSON.stringify({ data: 'ok' }));
  } else {
    res.statusCode = 404;
    res.end();
  }
}).listen(3000);

浏览器测试：在 http://localhost:8080 页面发起 fetch('http://localhost:3000/api/data', { method: 'PUT', headers: { 'Content-Type': 'application/json' } })，会先看到OPTIONS请求，再看到PUT请求。若服务端不返回Access-Control-Allow-Origin，浏览器将拦截响应。
```

### 4. 常见误区与进阶思考
误区1：认为CORS是服务端安全机制。实际上，服务端只是返回响应头，并不强制拦截任何请求。任何HTTP客户端（如curl）都能直接跨域请求，不会被CORS限制。CORS的“拦截”完全由浏览器执行，且只拦截响应，不拦截请求发送。若服务端依赖CORS做安全防护，等于形同虚设。

误区2：将Access-Control-Allow-Origin设置为*视为安全配置。当*与Access-Control-Allow-Credentials: true同时出现时，浏览器会拒绝该响应（规范禁止）。但即使不携带凭证，如果服务端反射任意Origin，任何恶意网站都可以通过浏览器向该服务端发起请求并读取响应，这等于完全开放了资源访问。正确做法是维护一个可信Origin白名单，仅反射白名单中的Origin，并验证其合法性。

思考题：若服务端将Access-Control-Allow-Origin设置为动态反射，并同时设置Access-Control-Allow-Credentials: true，且业务接口依赖Cookie验证身份。请说明浏览器如何执行CORS检查，以及这种配置为何能导致用户数据被恶意网站窃取？应如何修正？
