---
title: "每日基础技术总结 · 2026-08-19 · 跨域的本质与解决方案"
date: 2026-08-19 18:34:21
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-08-19 · 跨域的本质与解决方案

## 📚 今日主题

> **跨域的本质与解决方案**（前端底层与计算机基础）

### 1. 核心概念速览
跨域（Cross-Origin）的本质是浏览器同源策略（Same-Origin Policy, SOP）对跨源资源读取的默认禁止。同源由 URI 三元组（scheme, host, port）唯一确定，任一维度不同即构成跨域。SOP 是浏览器内建的安全隔离机制，将资源操作分为三类：写入（导航、表单提交）默认放行，嵌入（<script>、<img>、<iframe>）默认放行，读取（fetch/XHR 响应体、iframe DOM、Cookie）默认禁止。它解决的问题是：防止恶意源在用户已认证的会话上下文（Cookie、OAuth token）下跨源读取第三方资源的敏感数据。其机制本质是浏览器充当引用监控器（Reference Monitor），在每次跨源读取决策点检查目标源是否显式授权；授权协议即 CORS（Cross-Origin Resource Sharing），通过 HTTP 响应头声明允许的源集合。该知识点位于 Web 安全模型的基座，与 CSRF 防护、XSS 防护并列为浏览器三大安全防线；它不属于 HTTP 协议本身，而是浏览器安全上下文的访问控制策略，CORS 才是 HTTP 层面的协商协议。专业工程师必须掌握它：所有 B/S 架构的数据通信、认证体系（Cookie/Session/Token）、微前端隔离、iframe 协作均受其约束；后端与 AI 工程师在提供 API 时，若不理解 SOP 与 CORS 的边界，将无法解释“请求已到达服务器却被浏览器拦截”的现象，也无法设计正确的鉴权与跨域策略。注意：SOP 仅存在于浏览器环境，curl、Node.js、服务端间调用天然不受其约束——这也是为什么 AI 服务间调用不存在跨域问题。

### 2. 底层原理剖析
SOP 的执行粒度是源而非页面。判定跨源后，对不同操作执行不同策略：写入默认允许（<a> 跳转、<form> 提交），响应在新上下文呈现；嵌入默认允许（<img>、<script>、<link>、<iframe>），但嵌入方无法读取被嵌入资源的内容；读取默认禁止，必须经 CORS 协商。

CORS 协商流程（伪代码）：

处理跨源请求(req):
  origin = req.headers.Origin
  若 origin 为空 或 origin 等于目标源:
    正常处理（同源或非浏览器客户端）
  否则:
    若 req 需要预检(req.method, req.headers):
      发送 OPTIONS 至目标 URL，携带:
        Origin: origin
        Access-Control-Request-Method: req.method
        Access-Control-Request-Headers: req.headers 的键集合
      若预检响应缺少匹配 origin 的 Access-Control-Allow-Origin:
        抛 CORS 错误，真实请求不发送
      若 req.method 不在 Allow-Methods 或请求头不在 Allow-Headers 中:
        抛 CORS 错误，真实请求不发送
    发送真实请求
    检查响应头 Access-Control-Allow-Origin:
      若等于 origin，或等于 * 且请求未携带凭证:
        将响应体暴露给 JS
      否则:
        丢弃响应体，抛 TypeError（注意：请求已在服务器产生副作用）

触发预检的条件：方法非 GET/HEAD/POST；或设置了非简单请求头（Content-Type 非 x-www-form-urlencoded / multipart/form-data / text/plain 三者之一、Authorization、任何自定义头）；或请求体为 ReadableStream。

解决方案的本质分类：
1. 消除跨域——反向代理：浏览器与代理同源，代理将请求转发至目标后端（旧方案 document.domain 降级同源范围已废弃，仅作历史认知）。适用于无法修改后端响应头或需隐藏内部拓扑的场景。
2. 利用 SOP 嵌入豁免权——JSONP（<script> 加载并执行回调，仅支持 GET，信任模型为无条件信任目标源）、WebSocket（不受 SOP 约束，握手阶段由服务端校验 Origin）、postMessage（跨窗口消息通信，接收方必须显式校验 event.origin）。
3. CORS 协商放行——服务器通过 Access-Control-Allow-* 系列头显式声明授权边界。这是唯一标准的协议级方案，且与凭证模式存在强约束（详见误区）。

与前端已有概念的对比（类比 Java 接口与 TS 接口的本质差异）：Java 接口是运行期契约，由 JVM 在类加载与调用时强制校验；TS 接口是编译期结构，编译为 JS 后彻底消失。同理，CORS 与 JSONP 表面都是“跨域读取方案”，本质截然不同——CORS 是运行期的显式协议协商，浏览器对每一次响应强制检查头字段，信任模型是“目标服务器明确授权”；JSONP 是加载期的结构性旁路，利用 <script> 的嵌入豁免权将数据封装为可执行 JS，信任模型是“调用方无条件信任目标服务器”，且无法使用非 GET 方法、无法携带自定义头。这组对比说明：命名相近的方案，其底层机制与信任边界可能完全不同，必须从协议层分辨。

### 3. 基础代码与实战验证
```text
// server.js — 原生 Node.js HTTP 服务器，零框架验证 CORS 协商机制
const http = require('http');

http.createServer((req, res) => {
  // 浏览器在每次跨源请求中自动附加 Origin 头，这是协商的起点
  const origin = req.headers.origin;

  // 分支一：预检请求。浏览器以 OPTIONS 探测服务器的授权边界
  if (req.method === 'OPTIONS') {
    res.writeHead(204, {
      'Access-Control-Allow-Origin': origin,          // 回显具体源；凭证模式下不能用 *
      'Access-Control-Allow-Methods': 'GET,PUT',      // 方法白名单，决定真实请求是否放行
      'Access-Control-Allow-Headers': 'Content-Type', // 自定义头白名单，浏览器逐一比对
      'Access-Control-Max-Age': '3600'                // 预检结果缓存时长，减少 OPTIONS 次数
    });
    res.end();
    return;
  }

  // 分支二：真实请求。响应头必须再次声明授权，否则浏览器丢弃响应体
  res.writeHead(200, {
    'Access-Control-Allow-Origin': origin,            // 每次响应都要携带，非一次性配置
    'Access-Control-Allow-Credentials': 'true'        // 允许 JS 读取携带凭证的响应
  });
  res.end(JSON.stringify({ code: 0, data: 'ok' }));
}).listen(3000);

// browser.js — 在 http://localhost:8080 页面中执行
// 场景 A：PUT + JSON 请求，必然触发预检
fetch('http://localhost:3000/api', {
  method: 'PUT',
  headers: { 'Content-Type': 'application/json' },
  credentials: 'include',            // 要求携带 Cookie，启用凭证模式
  body: JSON.stringify({ a: 1 })
}).then(r => r.json()).then(console.log);

// 场景 B：注释 server.js 分支二的 Access-Control-Allow-Origin 头后重试
// 现象：真实请求已到达服务器（服务器日志可见），
// 但浏览器抛 CORS policy 错误，响应体被丢弃。
// 证明：SOP 拦截的是“读取”而非“发送”。

// 场景 C：curl -X PUT http://localhost:3000/api
// 现象：无任何跨域错误。证明 SOP 是浏览器行为，非服务器或协议强制。

验证步骤：依次执行场景 A/B/C，观察浏览器 Network 面板中的 OPTIONS 记录、服务器日志与 console 报错，可完整还原预检、放行、拦截三个阶段的底层行为。
```

### 4. 常见误区与进阶思考
误区一：将“跨域报错”理解为“请求被浏览器阻止”，并将跨域归为后端配置问题。事实：对于简单请求，浏览器在报错前已将请求完整发出，服务器端副作用已发生；CORS 拦截发生在响应阶段，浏览器丢弃的是响应体而非请求。因此 CORS 的本质是“服务器授权浏览器读取响应”的协商协议，是前后端协作的访问控制，而非单一端的开关。排障时必须先在服务端确认请求是否到达，否则会误判问题层次。

误区二：认为“Access-Control-Allow-Origin: *”是通用解。事实：当请求携带凭证（credentials: 'include' 或同源 Cookie）时，浏览器强制要求 Allow-Origin 为精确源且必须同时返回 Access-Control-Allow-Credentials: true；通配符会被直接拒绝。此外，* 意味着任意源均可读取响应，对携带敏感信息的接口等同于数据公开，属严重的安全配置错误。

进阶思考题：攻击者在 evil.com 构造隐藏 <form>，向 victim-bank.com/api/transfer 提交 POST。该请求是简单请求，不触发预检；浏览器会自动附带 victim-bank.com 域下的 Cookie；且表单提交属于 SOP 默认放行的写入操作，无需任何 CORS 授权即可发出。银行服务器执行转账后返回 JSON，浏览器因 SOP 将响应体丢弃，攻击者无法读取。请回答：转账是否成功？SOP 在此场景中实际防御了什么？现代 Web 应用真正依赖什么机制阻止此类攻击？

答案方向：转账成功——SOP 只限制跨源读取，不限制跨源写入。SOP 的防御价值在于攻击者无法读取转账接口的响应，无法利用响应中的敏感数据（token、余额、订单号）进行后续攻击。真正防御此类攻击的是 CSRF Token 校验与 SameSite Cookie（Lax/Strict），后者从 Cookie 发送策略上阻断跨站请求携带凭证，使该跨源写入请求失去认证上下文，从而在根源上消除攻击面。
