---
title: "每日基础技术总结 · 2025-07-16 · 跨域 CORS：预检请求、凭证与同源策略本质"
date: 2025-07-16 20:00:00
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础（扩展）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2025-07-16 · 跨域 CORS：预检请求、凭证与同源策略本质

## 📚 今日主题

> **跨域 CORS：预检请求、凭证与同源策略本质**（前端底层与计算机基础（扩展））

### 1. 核心概念速览
CORS (Cross-Origin Resource Sharing) 并非独立的安全机制，而是浏览器对同源策略（Same-Origin Policy, SOP）施加的运行时限制的一种妥协方案。SOP 是浏览器引擎基于 URL 协议、主机名和端口三元组实施的原生安全隔离机制，旨在防止脚本访问不同源的资源以对抗 CSRF 等攻击。CORS 通过 HTTP 响应头（如 Access-Control-Allow-Origin）向浏览器暴露跨域许可权，解决后端服务需要被不同前端域调用的工程需求。在 AI 体系中，掌握 CORS 是理解模型服务 API 部署（通常微服务化导致域名/端口差异）以及数据管道安全边界的基石。对于专业工程师，理解其本质在于区分‘网络层面的通信’与‘脚本层面的读取’，前者由 TCP/IP 栈处理，后者受浏览器沙箱约束。

### 2. 底层原理剖析
1. 同源判定：浏览器在执行 XHR/Fetch 前，计算请求目标源的 Scheme+Host+Port 是否与当前文档源一致。若不一致，标记为跨域请求。
2. 简单请求流程：浏览器发送实际 HTTP 请求 -> 服务端响应 Headers 中包含 Access-Control-Allow-Origin。浏览器检查该头部，若匹配或为 '*' 且不含凭证，则解析 Response Body；否则拦截并抛出 TypeError。
3. 预检请求（Preflight）机制：当请求方法非 GET/HEAD/POST，或 Content-Type 非 application/x-www-form-urlencoded/multipart/form-data/text/plain，或包含自定义 Header 时，触发预检。浏览器先发送 OPTIONS 请求至目标资源，询问服务端是否允许后续的实际请求。只有当服务端在 OPTIONS 响应中返回正确的 Allow-Methods、Allow-Headers 且状态码为 2xx 时，浏览器才会放行随后的真实请求。
4. 凭证特殊性：当 credentials: 'include' 时，Access-Control-Allow-Origin 严禁使用通配符 '*'，必须指定具体的源。这是因为 Cookie/Authorization Header 涉及身份标识，通配符会导致凭据泄露风险。浏览器会额外校验 Origin 与响应头的精确匹配。
对比 TS Interface 与 Java Interface：TS Interface 是编译时静态类型检查，用于规范代码结构，不生成任何运行时代码；Java Interface 是运行时二进制契约的一部分。而 CORS Header 是运行时 HTTP 协议的元数据交互，它不是逻辑约束，而是能力声明，类似于 RPC 中的 Service Description，决定调用者能否获取数据而非调用本身是否合法。

### 3. 基础代码与实战验证
```text
const handler = (req, res) => {
  // 1. 统一设置来源白名单，此处演示具体源而非通配符以支持凭证
  const allowedOrigin = req.headers.origin;
  // 生产环境中应校验 allowedOrigin 是否在预配置的白名单数组内
  res.setHeader('Access-Control-Allow-Origin', allowedOrigin);
  
  // 2. 处理预检请求 (OPTIONS)
  if (req.method === 'OPTIONS') {
    res.setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE');
    res.setHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization');
    res.setHeader('Access-Control-Max-Age', '86400'); // 缓存预检结果的时间
    return res.status(204).end(); // 204 No Content，无 Body
  }
  
  // 3. 处理实际请求
  res.setHeader('Access-Control-Allow-Credentials', 'true'); // 明确允许携带 Cookie/Token
  res.json({ message: 'Success', data: 'secure payload' });
};
// 关键注释：
// - Access-Control-Allow-Origin: 浏览器仅据此判断是否向 JS 暴露 Response。
// - Access-Control-Allow-Credentials: 设为 true 后，* 将被拒绝，需显式指定 Origin。
// - OPTIONS 响应不包含业务数据，仅做权限握手。
```

### 4. 常见误区与进阶思考
['误区一：认为设置 CORS 头可以防御 CSRF 攻击。CORS 仅控制浏览器是否将响应内容暴露给 JavaScript，它不阻止浏览器发送请求本身。恶意网站依然可以携带用户凭证发起 POST 请求到受保护端点，因此 CSRF 防护仍需依赖 SameSite Cookie 属性或 Token 验证，而非 CORS。\\n误区二：混淆 Network Tab 中的请求与 JS 抛出的错误。开发者工具中可能看到跨域请求成功发出并收到 200 OK，但在控制台报 Cross-Origin Request Blocked。这是因为浏览器在网络层完成了通信，但在应用层（Script Execution）依据 SOP 拒绝了数据读取，这是浏览器内核的安全剪断操作。']
