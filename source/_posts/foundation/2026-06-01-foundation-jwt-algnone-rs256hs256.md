---
title: "每日基础技术总结 · 2026-06-01 · JWT 的 alg=none 与 RS256/HS256 混淆攻击"
date: 2026-06-01 08:00:00
categories: [技术分享]
tags: ["技术分享", "安全基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-06-01 · JWT 的 alg=none 与 RS256/HS256 混淆攻击

## 📚 今日主题

> **JWT 的 alg=none 与 RS256/HS256 混淆攻击**（安全基础）

### 1. 核心概念速览
JWT（JSON Web Token）是一种自包含的令牌格式，由Base64URL编码的Header、Payload和Signature组成。Header中的alg字段声明了签名算法，例如RS256、HS256或none。alg=none表示令牌无签名，仅用于无认证场景；若验证端接受该值，攻击者可直接伪造任意token。RS256/HS256混淆攻击是指攻击者将Header中的alg从RS256改为HS256，并利用服务器公钥作为HMAC密钥重新签名，若服务器误将非对称算法公钥当作对称密钥进行验证，则可绕过签名校验。本质问题是：JWT的完整性验证依赖算法与密钥的严格匹配，而alg字段由客户端控制，服务端若未做算法白名单约束，信任边界被破坏。该知识点位于Web安全与认证授权领域，是API网关、微服务鉴权的基础。专业工程师必须掌握，因为它是实际生产环境中高频漏洞（如CVE-2019-7644）的根源，且理解它有助于正确设计签名验证流程。

### 2. 底层原理剖析
JWT验证的底层机制：服务端收到token后，从Header解析alg，根据该值选择对应算法和密钥进行签名验证。攻击路径1：alg=none。构造Header {alg:none}，Payload自定义，Signature置空。服务端若未禁用none，则直接跳过签名校验，只解析Payload即可通过。攻击路径2：算法混淆。合法流程：服务端用RSA私钥签名，公钥分发用于验证。攻击者将Header改为HS256，Payload保持不变，计算HMAC-SHA256(Payload, 公钥内容)作为新签名。由于HS256是对称算法，验证端需要用同一个密钥。若服务端误将公钥字符串当作HS256的共享密钥，则验证通过。这源于标准中alg字段决定验证行为，而密钥类型与算法不匹配时，某些库（如旧版jsonwebtoken）不会强制检查。对比前端概念：类似于前端表单提交时，后端信任了HTTP请求中的Content-Type字段，而该字段可被客户端任意修改。前端工程师熟悉「永远不要信任客户端输入」，但JWT的Header也是客户端可控的，且某些库的默认行为会信任alg。另一个对比：TypeScript的接口是编译期约束，而JWT的签名是运行期约束；编译期约束可以被绕过（只要运行时没有检查），类似地，JWT的alg声明如果没有被运行时强制执行，就是纸面约束。

### 3. 基础代码与实战验证
```text
以下用Node.js crypto库演示两种攻击的构造与验证绕过。

// 构造alg=none的token
const header = Buffer.from(JSON.stringify({alg:'none', typ:'JWT'})).toString('base64url');
const payload = Buffer.from(JSON.stringify({sub:'admin', iat: Date.now()})).toString('base64url');
const token = `${header}.${payload}.`; // 空签名

// 验证端若允许none，则直接通过
// 假设verifyToken函数先解析header.alg，若为none则不校验签名。

// 构造RS256->HS256混淆token
// 公钥内容公开，攻击者获得公钥字符串 publicKeyPem
const h = Buffer.from(JSON.stringify({alg:'HS256', typ:'JWT'})).toString('base64url');
const p = Buffer.from(JSON.stringify({sub:'admin', iat: Date.now()})).toString('base64url');
const signingInput = `${h}.${p}`;
const signature = crypto.createHmac('sha256', publicKeyPem).update(signingInput).digest('base64url');
const forgedToken = `${h}.${p}.${signature}`;

// 服务端错误验证方式：
// const decoded = jwt.verify(forgedToken, publicKeyPem, {algorithms:['RS256','HS256']});
// 如果库根据alg选择HS256，并将publicKeyPem作为secret，则签名通过。

// 正确做法：验证时必须明确指定算法白名单，且用公钥验证RS256时，需要检查alg。
// jwt.verify(forgedToken, publicKeyPem, {algorithms:['RS256']}); // 会抛出错误，因为alg是HS256。

注意：crypto.createHmac需要密钥为字符串或Buffer，publicKeyPem是PEM格式的公钥字符串，可直接使用。
```

### 4. 常见误区与进阶思考
误区1：认为JWT是加密的，所以内容安全。实际上JWT的Payload只是Base64URL编码，可被任何人解码，签名只保证内容未被篡改，不提供机密性。攻击者可读取和伪造内容，只要通过签名校验。
误区2：只验证签名而不校验alg，或依赖库的默认行为。很多库（如jsonwebtoken v8之前）默认接受none，或者当algorithms未指定时，允许任意alg。正确做法是始终明确指定允许的算法列表，并拒绝none。
思考题：如果服务端正确校验了alg为RS256，且使用公钥验证，攻击者还有哪些方式可能绕过签名？请从JWT的解析和密钥管理角度分析。提示：考虑算法歧义（如RS256与PS256）、密钥注入、公钥污染、或JWT头部中的kid参数导致路径遍历等。
