---
title: "每日基础技术总结 · 2026-06-12 · CSRF 的 SameSite Cookie 与双重提交令牌"
date: 2026-06-12 08:00:00
categories: [技术分享]
tags: ["技术分享", "安全基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-06-12 · CSRF 的 SameSite Cookie 与双重提交令牌

## 📚 今日主题

> **CSRF 的 SameSite Cookie 与双重提交令牌**（安全基础）

### 1. 核心概念速览
CSRF（跨站请求伪造）的本质是：浏览器在向目标站点发起请求时，会自动携带该站点的 Cookie（包括会话凭据），而目标站点无法区分请求是否由用户本人通过合法页面发出。SameSite Cookie 与双重提交令牌是两种不同层级的防御机制：前者是浏览器侧对 Cookie 发送策略的约束，后者是应用层通过加密随机令牌的一致性校验。SameSite 属性（Strict/Lax/None）直接规定跨站请求中是否携带 Cookie，从源头阻断携带会话凭据的跨站请求；双重提交令牌则利用同源策略下跨站页面无法读取目标站点 Cookie 的特性，要求请求中必须携带一个与 Cookie 中值相同的额外令牌，服务器校验二者一致。两者都解决同一问题，但作用点不同：SameSite 依赖浏览器实现，双重提交令牌依赖应用自校验。作为专业工程师，必须同时理解两者，因为实际攻击往往绕过单一机制，且二者在浏览器演进与微服务架构下有不同的失效场景。这一知识点位于 Web 安全基础、浏览器安全模型与 HTTP 协议的交汇处，是理解认证、授权、同源策略与 CSRF 攻击链的基石。

### 2. 底层原理剖析
SameSite Cookie 的底层机制基于“同站”（site）概念，而非“同源”（origin）。同站由注册域（eTLD+1）决定，不关心端口与协议。浏览器在 Set-Cookie 中收到 SameSite=Lax 时，会遵循以下规则：若请求是顶级导航（地址栏输入、链接跳转）且方法为 GET，则允许携带 Cookie；若是子资源请求（img/script/fetch/XHR）或非 GET 表单提交，则不允许携带。SameSite=Strict 则任何跨站请求都不携带。SameSite=None 必须配合 Secure 属性，否则被浏览器拒绝。注意：跨站与跨源不同，例如 a.example.com 与 b.example.com 是跨站，而 example.com:8080 与 example.com:9090 是同站但跨源。双重提交令牌（Double Submit Token）的机制是：服务器在渲染页面时生成一个不可预测的随机 Token，将其同时放入一个 Cookie（例如 csrf_token）和请求参数（或自定义头）。浏览器发出的任何跨站请求（表单、fetch）无法通过脚本读取目标站点的 Cookie 值（同源策略），因此攻击者无法在请求中构造出与 Cookie 一致的 Token。服务器校验请求中的 Token 与 Cookie 中的 Token 是否相等，相等则放行，否则拒绝。其伪代码如下：

服务端生成会话页面：
  token = random()
  Set-Cookie: csrf_token=token; SameSite=Lax; Path=/
  页面内嵌 JavaScript：
    const token = document.cookie.match(/csrf_token=([^;]+)/)[1]
    发送请求时设置 Header: X-CSRF-Token: token

服务端校验请求：
  cookieToken = req.cookies['csrf_token']
  headerToken = req.headers['X-CSRF-Token']
  if (!cookieToken || !headerToken || cookieToken !== headerToken) return 403
  处理业务逻辑

这里需要理解两种机制的对比：SameSite 是浏览器自动执行的安全策略，属于“平台约束”；双重提交令牌是应用层校验，属于“业务逻辑约束”。它们的关系类似前端中 Java 接口与 TypeScript 接口：Java 接口是编译期的名义类型约束，强制实现类必须声明对应方法，但运行时不保留接口信息；TS 接口是结构类型约束，只要对象形状匹配即可通过编译，但在运行时被擦除。两者都旨在保证“契约一致”，但约束的时机和位置完全不同。SameSite 在浏览器发起请求时就已经决定是否携带凭据，而双重提交令牌在请求到达服务器后才验证，二者形成纵深防御：即使 SameSite 被绕过（例如同站子域污染），双重提交令牌仍可能阻止攻击。

### 3. 基础代码与实战验证
```text
以下是一个基于 Node.js 原生 http 模块的极简示例，演示双重提交令牌的服务端逻辑，并设置 SameSite Cookie。浏览器端 JavaScript 会读取 Cookie 中的 Token，并放入自定义请求头。

const http = require('http');
const crypto = require('crypto');

const server = http.createServer((req, res) => {
  // 解析 Cookie 和请求头
  const cookies = req.headers.cookie || '';
  const cookieToken = cookies.split(';').map(s => s.trim()).find(s => s.startsWith('csrf_token='))?.split('=')[1];
  const headerToken = req.headers['x-csrf-token'];

  if (req.url === '/') {
    // 生成新 Token，并写入 Cookie（SameSite=Lax 限制跨站携带）
    const token = crypto.randomBytes(16).toString('hex');
    res.setHeader('Set-Cookie', `csrf_token=${token}; SameSite=Lax; Path=/; HttpOnly`);
    // 返回页面，页面脚本读取 Cookie 并放入请求头（注意 HttpOnly 会阻止脚本读取，因此演示用非 HttpOnly）
    // 这里为了演示双重提交，将 Cookie 设为非 HttpOnly，并返回一段 JS
    res.end(`
      <script>
        function getCookie(name) {
          const m = document.cookie.match(new RegExp('(^| )' + name + '=([^;]+)'));
          return m ? m[2] : '';
        }
        async function send() {
          const token = getCookie('csrf_token');
          await fetch('/submit', {
            method: 'POST',
            headers: { 'X-CSRF-Token': token },
          });
        }
        send();
      </script>
    `);
    return;
  }

  if (req.url === '/submit' && req.method === 'POST') {
    // 校验双重提交令牌：请求头中的 Token 必须与 Cookie 中的 Token 相等
    if (!cookieToken || !headerToken || cookieToken !== headerToken) {
      res.statusCode = 403;
      res.end('CSRF validation failed');
      return;
    }
    res.end('OK');
    return;
  }

  res.statusCode = 404;
  res.end('Not Found');
});

server.listen(3000);

注释：
- 第 8 行：从 Cookie 头中提取 csrf_token 值。
- 第 9 行：从 X-CSRF-Token 请求头提取令牌。
- 第 13 行：设置 SameSite=Lax，浏览器在跨站 POST 或 fetch 时不会携带该 Cookie，因此 cookieToken 为空。
- 第 15 行：如果 Cookie 设置了 HttpOnly，JavaScript 无法读取，双重提交会失败；所以真实场景需要权衡，或使用自定义头加表单字段。
- 第 28 行：校验请求头与 Cookie 的一致性，这是双重提交的核心。
```

### 4. 常见误区与进阶思考
误区一：认为 SameSite=Strict 可以彻底防御 CSRF。实际上，SameSite 只控制“跨站请求”是否携带 Cookie，但同站子域（例如 attacker.example.com 与 app.example.com）依然可以互相发送请求，且 SameSite=Lax 会允许顶级导航的 GET 请求携带 Cookie，如果服务端对 GET 请求有副作用（不符合 REST 规范）仍可能被利用。此外，SameSite 对超链接、预加载、浏览器的自动填充等场景的覆盖不完全，且某些旧浏览器不支持。因此不能只依赖 SameSite，必须保留应用层校验。

误区二：双重提交令牌中把 Token 放在 Cookie 里并认为非 HttpOnly 是安全的，因为“跨站脚本无法读取”。实际上一旦存在任意 XSS，攻击者可以读取 Cookie 中的 Token 并构造请求，双重提交直接失效。另外，如果子域可以设置 Cookie（例如 Domain=.example.com），攻击者可以覆盖 csrf_token 的值为自己已知的 Token，从而绕过校验。双重提交令牌必须配合 Host-Only Cookie、Secure 属性和严格的子域隔离，且不应取代输出编码与 CSP。

深度思考题：如果攻击者能在受害者的浏览器中通过 JavaScript 读取目标站点的 Cookie（例如同源策略被绕过，或存在 XSS），但无法读取请求头的内容，那么双重提交令牌是否依然有效？请从同源策略、Cookie 的可访问性以及请求头的不可构造性三个角度分析其失效条件。
