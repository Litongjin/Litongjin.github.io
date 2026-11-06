---
title: "每日基础技术总结 · 2026-11-06 · HTTP Cookie的HttpOnly、Secure与SameSite属性安全模型"
date: 2026-11-06 10:52:17
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-06 · HTTP Cookie的HttpOnly、Secure与SameSite属性安全模型

## 📚 今日主题

> **HTTP Cookie的HttpOnly、Secure与SameSite属性安全模型**（后端基础）

### 1. 核心概念速览
Cookie 是 HTTP 的状态管理机制：服务器通过 Set-Cookie 响应头向浏览器 cookie store 写入键值对，浏览器在后续请求中按 domain/path/secure/samesite 等匹配规则自动附加 Cookie 请求头。HttpOnly、Secure、SameSite 是三个正交安全属性：HttpOnly 将 cookie 从 document.cookie 等脚本可见集合中移除；Secure 限定仅通过安全传输层回传；SameSite 以 site（scheme + registrable domain）为粒度限制跨站请求携带。其本质是把 cookie 这种环境权限（ambient authority）从无条件自动附加，改为按脚本可见性、传输通道、请求上下文三个维度判定。它位于浏览器 HTTP 客户端与 Web 应用会话层之间，是会话认证、XSS/CSRF 防御、CORS credentials、第三方 cookie 治理的基础。专业工程师必须掌握，因为认证 cookie 一旦可被脚本读取、明文传输或被跨站自动附加，后端鉴权边界即被绕过。

### 2. 底层原理剖析
1. 写入与存储
Set-Cookie: sid=...; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=3600
浏览器解析后存入 cookie jar，条目至少包含 name、value、domain、path、host-only、secure-only、http-only、same-site、expiry、partition key。HttpOnly 是存储条目上的布尔标志，不参与请求匹配，只影响脚本读取 API。

2. 请求匹配与附加
浏览器收到请求后遍历 cookie jar，按以下顺序过滤：domain-match（Host 与 cookie domain 后缀匹配且非公共后缀）、path-match（请求路径以 cookie path 为前缀）、Secure（cookie.secure 为真时请求 scheme 必须为 https，现代浏览器还拒绝在非安全源写入 Secure cookie）、SameSite（比较请求上下文与 cookie 的 site）。全部通过后拼成 Cookie: k=v; k2=v2 请求头。

3. HttpOnly
设置后 document.cookie 的 getter 不返回该 cookie；fetch/XHR 也无法通过 JS API 读取其值。但浏览器仍会自动附加到同源请求；XSS 仍可调用 fetch('/api', {credentials:'include'}) 发起带凭据请求。因此 HttpOnly 只切断“脚本读取 cookie 值”这一条路径，不阻断 XSS 的请求能力。

4. Secure
Secure 只约束传输通道：cookie 不会在 http:// 请求中回传；现代浏览器拒绝从非安全源设置 Secure cookie，防止网络攻击者用明文响应注入。它不保证 cookie 在服务器端不被覆盖，也不替代 HSTS、__Host- 前缀与输出编码。

5. SameSite
SameSite 判定基于 site，而非 origin：site = scheme + registrable domain（eTLD+1），端口忽略；origin = scheme + host + port。same-origin 必 same-site，same-site 不一定 same-origin。Strict：任何跨站请求都不携带；Lax：跨站顶层导航且安全方法（GET/HEAD）可携带，跨站子资源与跨站 POST 不携带；None：跨站也携带，但必须同时 Secure。现代浏览器默认 Lax。SameSite=None 受第三方 cookie 策略影响，可能被浏览器直接阻断。

伪代码：
sameSiteAllows(mode, req):
  if mode == None: return true
  if req.isSameSite: return true
  if mode == Strict: return false
  if mode == Lax: return req.isTopLevelNavigation && (req.method == GET || req.method == HEAD)
  return false

6. 与前端概念对比
localStorage/sessionStorage 是 JS 可读、按 origin 隔离、不会自动进入 HTTP 请求头；cookie 是浏览器自动附加的环境权限，HttpOnly 使其对 JS 不可见，SameSite 使其按请求上下文附加。CORS 控制的是跨源响应能否被 JS 读取；SameSite 控制的是跨站请求是否携带 cookie。CORS 配置错误不会直接产生 CSRF，SameSite 也不能赋予 JS 读取响应体的能力。Origin/Referer 校验是服务端收到请求后的判定，SameSite 是浏览器发送前的判定，二者互补。

### 3. 基础代码与实战验证
```text
const http = require('http');

const server = http.createServer((req, res) => {
  if (req.url === '/login') {
    // HttpOnly: 该 cookie 不进入 document.cookie 可见集合，JS 无法读取其值
    // Secure: 浏览器仅在 https 请求中回传；本地验证需 HTTPS 或受信 localhost
    // SameSite=Lax: 跨站子资源/跨站 POST 不携带；跨站顶层 GET 导航可携带
    res.setHeader('Set-Cookie', 'sid=abc123; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=3600');
    res.end('login ok');
    return;
  }

  if (req.url === '/me') {
    // 服务端只看到 HTTP 请求头 Cookie；HttpOnly 不改变浏览器自动附加行为
    const cookieHeader = req.headers.cookie || '';
    res.setHeader('Content-Type', 'application/json; charset=utf-8');
    res.end(JSON.stringify({ cookieHeader }));
    return;
  }

  res.setHeader('Content-Type', 'text/html; charset=utf-8');
  // 前端验证：document.cookie 读不到 HttpOnly cookie；fetch 仍可携带凭据访问同源接口
  res.end(`<script>document.write('document.cookie=' + document.cookie); fetch('/me',{credentials:'include'}).then(r=>r.text()).then(console.log)</script>`);
});

server.listen(3000, () => console.log('http://localhost:3000/login'));

// 浏览器验证步骤：
// 1. 访问 /login 写入响应头 Set-Cookie
// 2. 页面脚本执行 document.cookie，HttpOnly 值不可见
// 3. fetch('/me',{credentials:'include'}) 仍携带 Cookie 请求头，服务端可读到 sid=abc123
// 4. 用 curl -i http://localhost:3000/login 观察 Set-Cookie；curl 不执行浏览器 cookie 策略，仅用于观察响应头
// 5. 验证 Secure 必须使用 HTTPS 或浏览器认定的受信 localhost，否则浏览器会拒绝写入/回传
```

### 4. 常见误区与进阶思考
误区一：HttpOnly 等于 XSS 防护。本质：HttpOnly 只让 document.cookie 不返回该 cookie，阻断凭据值外泄；XSS 仍可在页面上下文发起同源请求、读取 DOM、调用 fetch 携带 cookie、执行用户权限操作。防御需 CSP、输出编码、Trusted Types、最小权限。
误区二：SameSite 是 CSRF 的完整解，或 CORS 能替代 SameSite。本质：SameSite 以 site 为粒度，同一 registrable domain 下的子域视为 same-site；攻击者控制子域时可绕过跨站判定。GET 状态变更、顶层导航、浏览器默认值与第三方 cookie 策略仍留边界。CORS 是响应读取授权，不是请求携带授权；CSRF token、Origin/Referer 校验、__Host- 前缀、HSTS 仍需组合。
思考题：设攻击者控制 evil.example.com，目标为 app.example.com。app.example.com 的会话 cookie 为 Domain=example.com; Path=/; HttpOnly; Secure; SameSite=Lax。攻击者从 evil.example.com 发起对 app.example.com 的跨源 fetch 或顶层导航时，SameSite 是否拦截？HttpOnly 与 Secure 分别能否阻止攻击者读取或覆盖该 cookie？请结合 cookie domain-match、same-site 判定、Secure 写入规则与浏览器第三方 cookie 策略分析。
