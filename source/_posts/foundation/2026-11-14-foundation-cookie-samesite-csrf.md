---
title: "每日基础技术总结 · 2026-11-14 · HTTP Cookie属性SameSite=Lax/Strict/None的安全性与跨站请求伪造（CSRF）防护"
date: 2026-11-14 07:01:47
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-14 · HTTP Cookie属性SameSite=Lax/Strict/None的安全性与跨站请求伪造（CSRF）防护

## 📚 今日主题

> **HTTP Cookie属性SameSite=Lax/Strict/None的安全性与跨站请求伪造（CSRF）防护**（网络基础）

### 1. 核心概念速览
核心概念速览：

SameSite 是 Cookie 属性，通过 Set-Cookie 设置，决定浏览器是否把 Cookie 带入“跨站”请求。它处理的不是认证本身，而是“凭证是否被自动附带”。Cookie 会话的本质是浏览器把身份凭证自动带到同站请求；CSRF 正是利用这一点：受害者已登录站点 B，攻击者站点 A 诱导浏览器向 B 发起携带凭证的状态变更请求，服务端误以为请求来自真实用户。SameSite 的作用是在请求发出前裁剪 Cookie：Strict 禁止跨站携带；Lax 仅允许顶层导航 GET 携带；None 允许跨站携带，但必须 Secure，并且受第三方 Cookie 策略限制。它在计算机体系里处于 HTTP 层、浏览器安全模型、会话状态模型之间：HTTP 传输 Header，浏览器维护 Cookie Jar，服务端用会话状态做鉴权。对后端、平台、安全、AI 服务调用同样重要，因为任何“浏览器自动携带身份”的系统都有被 CSRF 利用的通道。专业工程师必须掌握，否则会把“已登录”误当成“请求者真实意图”，把 CORS、Cookie、身份认证混为同一层。

### 2. 底层原理剖析
基础语义：
浏览器按 Name / Domain / Path / Secure / SameSite 存储 Cookie。请求前区分两个概念：Origin 是 scheme + host + port，Site 是 eTLD+1，例如 app.example.com 与 api.example.com 属于 same-site。因此 SameSite 判断通常基于 Site，而不是 Origin；http 与 https、端口差异在 SameSite 语义上不一定等价于跨站。

选择过程：
浏览器收到 Set-Cookie: session=...; SameSite=Lax; HttpOnly; Secure 后，存储规则为：
Secure = 仅 HTTPS 发送；
HttpOnly = JS 不可读；
Path/Domain 决定作用域；
SameSite 决定跨站请求是否附带。

策略伪代码：
function canSendCookie(request, cookie):
    if !cookie.matchPath(request.path):
        return false
    if !cookie.matchDomain(request.host):
        return false
    if cookie.secure and !request.isHttps:
        return false

    if isSameSite(request.origin, request.destination):
        return true

    switch cookie.sameSite:
        case Strict:
            return false
        case Lax:
            return request.isTopLevelNavigation and
                   request.method == GET
        case None:
            return browserAllowsThirdPartyCookie

三种模式解释：
Strict：完全禁止跨站携带。跨站 fetch、XHR、表单 POST、iframe、顶层导航 GET 都不会带 Cookie。安全边界更紧，但容易破坏合法跳转，例如邮件链接、OAuth 回调、第三方登录、站内跳转，甚至部分浏览器扩展/代理场景。
Lax：默认常用。跨站非导航请求不带 Cookie；顶层导航 GET 带 Cookie。它能挡住“表单 POST 状态修改”，但挡不住“GET 请求里偷偷修改状态”。例如受害者点击 https://bank.example/transfer?target=attacker&amount=100，浏览器仍然附带 Cookie，只要服务端把 GET 当业务处理，就会被 CSRF。
None：跨站携带，必须 Secure。它本质是放开跨站 Cookie，用于 iframe、跨站登录、跨域嵌入等场景。None 不是 CSRF 防护，反而把 CSRF 面扩大到请求发起层；需要额外 CSRF Token、Origin/Referer 校验、二次确认、状态变更只用 POST/PUT/DELETE、敏感操作绑定一次性凭证。

与前端体系关系：
TS interface 是编译期结构契约，关注类型是否满足；Java interface 是运行期行为契约，关注实现与多态。SameSite 不是类型语言契约，而是浏览器运行期策略契约，关注请求是否允许携带身份 Cookie。CORS 控制跨域请求发出与响应可读；SameSite 控制 Cookie 是否附带。二者不同：CORS 失败可能阻止读取，但表单导航类请求仍可能到达服务端；SameSite 不阻止导航请求本身，只决定 Cookie 是否随请求发出。

### 3. 基础代码与实战验证
```text
// target server: node target.js
// hosts 必须模拟两个不同 eTLD+1：127.0.0.1 target.dev 与 127.0.0.1 attacker.dev
// 测试路径：先访问 target.dev 登录，设置目标 Cookie；再打开 attacker.html 发起跨站请求。
// None 的跨站携带必须配合 HTTPS；本示例使用 HTTP，仅验证 Lax/Strict 主逻辑。

const http = require('http');

const server = http.createServer((req, res) => {
  const cookie = req.headers.cookie || '';

  if (req.url.startsWith('/login')) {
    const u = new URL(req.url, 'http://target.dev:8081');
    const mode = u.searchParams.get('mode') || 'lax';

    let attrs = 'HttpOnly; Path=/';

    // Strict：任何跨站请求均不携带，包括顶层导航。
    if (mode === 'strict') {
      attrs += '; SameSite=Strict';
    }

    // Lax：跨站 POST/XHR/fetch/iframe 不携带；顶层导航 GET 携带。
    if (mode === 'lax') {
      attrs += '; SameSite=Lax';
    }

    // None：跨站携带，但必须 Secure；生产 HTTPS 验证。
    if (mode === 'none') {
      attrs += '; SameSite=None; Secure';
    }

    // 关键：Set-Cookie 只是浏览器策略入口，真正防护在请求阶段由浏览器裁剪 Cookie。
    res.setHeader('Set-Cookie', 'sid=secret;' + attrs);

    res.statusCode = 204;
    res.end();
    return;
  }

  if (req.url.startsWith('/transfer') && req.method === 'POST') {
    // 服务端仅凭 Cookie 判断已登录，这是典型“认证不等于请求源验证”。
    // 当 SameSite=Lax/Strict 时，跨站表单/fetch POST 不会带 sid=secret，因此返回 401。
    const authed = cookie.includes('sid=secret');

    res.statusCode = authed ? 200 : 401;
    res.end(authed ? 'POST_STATE_CHANGE_EXECUTED' : 'NO_COOKIE');
    return;
  }

  if (req.url.startsWith('/transfer') && req.method === 'GET') {
    // 反例：GET 执行状态变更。Lax 下跨站顶层导航仍会带 Cookie。
    const authed = cookie.includes('sid=secret');

    res.statusCode = 200;
    res.end(authed ? 'GET_STATE_CHANGE_EXECUTED' : 'GET_NO_COOKIE');
    return;
  }

  res.statusCode = 404;
  res.end();
});

server.listen(8081, () => {
  console.log('target listening http://target.dev:8081');
});

// attacker.html 放在 http://attacker.dev:8082
/*
<form action='http://target.dev:8081/login?mode=lax' method='GET'>
  <button>set lax</button>
</form>

<form action='http://target.dev:8081/transfer' method='POST'>
  <button>POST CSRF</button>
</form>

<a href='http://target.dev:8081/transfer?target=1'>GET top-level navigation CSRF</a>

<button onclick='fetch("http://target.dev:8081/transfer", { method: "POST", credentials: "include" })'>fetch CSRF</button>
*/
```

### 4. 常见误区与进阶思考
容易踩坑：
1. 误把 SameSite 当完整 CSRF 方案。
Lax 挡不住跨站顶层导航 GET；如果 GET 里有转账、删除、权限修改，攻击者只要诱导点击 https://target.example/change?role=admin 就能完成 CSRF。Strict 能挡更多跨站场景，但会破坏合法跳转、OAuth、iframe、邮件链接等场景。正确原则是：状态变更只能使用非 GET 方法；写操作必须做请求源验证与业务确认，而不是依赖浏览器是否带 Cookie。

2. 误以为 SameSite=None 是“更安全/更现代”。
None 的本质是允许跨站携带，必须 Secure，但仍不能阻止 CSRF。None 常用于复杂跨域交互，但风险更高；必须搭配 CSRF Token、Origin/Referer 校验、一次性确认、敏感操作二次认证。否则它等于把 Cookie 暴露给所有被允许的跨站请求。

进阶思考题：
如果某接口 GET /api/delete?id=123 返回 200 并删除资源，SameSite=Lax 能否阻止攻击？答案是不能。因为 Lax 允许顶层导航 GET 携带 Cookie，受害者只要访问攻击者页面里的 /api/delete?id=123 链接，请求就是合法同站 GET，服务端仍会收到删除请求。这说明 Cookie 只解决“凭证是否附带”，不解决“谁授权发起这个状态变更”。
