---
title: "每日基础技术总结 · 2026-09-30 · JWT 结构、签名验证与常见攻击"
date: 2026-09-30 07:05:48
categories: [技术分享]
tags: ["技术分享", "Java 后端与 Spring 生态"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-30 · JWT 结构、签名验证与常见攻击

## 📚 今日主题

> **JWT 结构、签名验证与常见攻击**（Java 后端与 Spring 生态）

### 1. 核心概念速览
JWT（JSON Web Token，RFC 7519）是一种无状态、自包含的认证凭据载体，由三段 Base64Url 编码的字符串组成：Header.Payload.Signature。其本质是：服务端用密钥对声明（Claim）进行签名，客户端持有该令牌并在后续请求中携带，服务端通过重新计算签名并比对来验证令牌的完整性与可信来源。它解决的是分布式系统中认证信息的「可信传递」问题，避免每次请求都查询会话存储，从而支持水平扩展、跨域认证与微服务间信任传递。在整个计算机体系中，JWT 属于应用层认证机制，位于 HTTP 之上，与 TLS 解决传输加密、OAuth2 解决授权流程、Session 解决服务端状态存储的层次不同。专业工程师必须掌握它，因为几乎所有现代后端框架、API 网关、单点登录系统都原生支持 JWT，且安全漏洞（如算法混淆、密钥泄露）直接导致账户接管，属于高危安全事故。

### 2. 底层原理剖析
JWT 的签名验证基于非对称或对称密码学。Header 指定签名算法（如 HS256、RS256），Payload 包含注册声明（iss、exp、sub 等）和自定义声明。签名生成过程为：

```
signature = HMAC_SHA256( base64url(header) + '.' + base64url(payload), secret )  // HS256
或
signature = RSA_SHA256( base64url(header) + '.' + base64url(payload), privateKey ) // RS256
```

验证时，服务端使用相同算法和密钥（对称时用同一 secret，非对称时用公钥）对前两段重新计算签名，并与第三段进行恒定时间比较（防止时序攻击）。若不一致，则令牌被篡改；若一致，还需校验 exp（过期时间）、nbf（不早于）、iss（签发者）等声明，防止重放和跨域使用。

底层机制的本质是「签名=对内容的哈希加密钥处理」，因此签名只能证明内容是自洽且由密钥持有者签发，不能加密内容（Payload 可被任何人 Base64 解码）。这与前端工程中「接口（Interface）是编译期结构约束」有本质区别：TypeScript 的 interface 只是类型系统内的形状约定，在运行时完全擦除；而 JWT 的签名是运行时的密码学约束，保证数据未被篡改且来源可信。另一个类比是浏览器 Cookie 中的 HttpOnly + SameSite 属性，它们是通过浏览器实施的安全边界，而 JWT 签名是通过数学算法实施的安全边界，二者都需要防 XSS 与 CSRF 攻击。

### 3. 基础代码与实战验证
以下用 Node.js 的 crypto 模块实现最简 HS256 JWT 签名与验证，不依赖任何框架，展示底层原理。

```js
const crypto = require('crypto');

// Base64Url 编码（JWT 使用 URL 安全字符集，并去除填充'='）
function b64url(input) {
  return Buffer.from(JSON.stringify(input))
    .toString('base64')
    .replace(/=/g, '')
    .replace(/\+/g, '-')
    .replace(/\//g, '_');
}

function b64urlDecode(str) {
  str = str.replace(/-/g, '+').replace(/_/g, '/');
  while (str.length % 4) str += '=';
  return Buffer.from(str, 'base64').toString('utf8');
}

// 关键：签名 = HMAC-SHA256(header.payload, secret)
function sign(header, payload, secret) {
  const data = `${b64url(header)}.${b64url(payload)}`;
  const hmac = crypto.createHmac('sha256', secret);
  hmac.update(data);
  const sig = hmac.digest('base64').replace(/=/g, '').replace(/\+/g, '-').replace(/\//g, '_');
  return `${data}.${sig}`;
}

// 关键：验证 = 用同一密钥重算签名，并与传入签名比对（需恒定时间比较）
function verify(token, secret) {
  const [h, p, s] = token.split('.');
  const header = JSON.parse(b64urlDecode(h));
  if (header.alg !== 'HS256') throw new Error('Unsupported alg'); // 必须白名单校验算法，防止 alg=none 攻击
  const data = `${h}.${p}`;
  const expected = crypto.createHmac('sha256', secret).update(data).digest('base64')
    .replace(/=/g, '').replace(/\+/g, '-').replace(/\//g, '_');
  // 使用 timingSafeEqual 避免时序攻击，这里简化为字符串比较（生产务必用安全比较）
  if (expected !== s) throw new Error('Invalid signature');
  const payload = JSON.parse(b64urlDecode(p));
  if (payload.exp && Date.now() >= payload.exp * 1000) throw new Error('Expired'); // 验证过期
  return payload;
}

// 演示：签发 60 秒有效的 token
const payload = { user: 'alice', exp: Math.floor(Date.now()/1000) + 60 };
const token = sign({alg:'HS256', typ:'JWT'}, payload, 'my-secret');
console.log(token);
console.log(verify(token, 'my-secret')); // { user: 'alice', exp: ... }
```

生产环境中应使用如 jose、jsonwebtoken 等库，但底层原理与以上完全一致：先拼接 header.payload，再用密钥计算 HMAC，最后比对签名并校验声明。

### 4. 常见误区与进阶思考
误区1：认为 JWT 的 Payload 是加密的。实际 Payload 仅 Base64Url 编码，任何人可解码读取。不能在 JWT 中存放密码、身份证号等敏感信息。解决方式：只存必要声明，敏感数据通过引用（如数据库 ID）放在服务端。

误区2：验证时只用签名算法，不校验算法类型。攻击者可将 Header 中 alg 改为 'none' 或改为 'HS256'（当服务端用 RS256 时，攻击者用服务器公钥作 HMAC secret 就能伪造）。必须显式白名单算法，并严格匹配密钥类型。

思考题：假设某服务端使用 RS256（私钥签名，公钥验证），但同时接受 HS256。攻击者获取了公钥（公开的），他将 JWT 的 header 改为 alg=HS256，payload 改为任意内容，然后使用公钥字符串作为 HMAC secret 计算签名。服务端验证时会用公钥字符串作为 HS256 密钥重新计算，签名一致，从而伪造任意 token。如何从代码层面彻底阻断这种攻击？（提示：分别校验 alg 是否为白名单，以及在解析 JWT 时绑定预期算法与密钥用途。）
