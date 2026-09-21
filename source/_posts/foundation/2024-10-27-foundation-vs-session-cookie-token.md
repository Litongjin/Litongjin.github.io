---
title: "每日基础技术总结 · 2024-10-27 · 认证 vs 授权：Session/Cookie 与 Token"
date: 2024-10-27 20:00:00
categories: [技术分享]
tags: ["技术分享", "安全进阶（Web / 认证授权 / 密码学）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-10-27 · 认证 vs 授权：Session/Cookie 与 Token

## 📚 今日主题

> **认证 vs 授权：Session/Cookie 与 Token**（安全进阶（Web / 认证授权 / 密码学））

### 1. 核心概念速览
认证（Authentication）与授权（Authorization）是安全体系的两大支柱。认证解决‘你是谁’的身份核验问题，通过凭证比对确认实体身份；授权解决‘你能做什么’的权限控制问题，基于身份映射资源访问策略。Session/Cookie 是无状态协议（HTTP）下由服务端维持状态的会话管理方案，核心机制是 Session ID 在服务端存储、Cookie 在客户端传递；Token（如 JWT）是无状态自包含的令牌，将用户声明（Claims）签名后直接嵌入消息体，服务端无需本地状态即可验证。在现代分布式系统与微服务架构中，Token 因其无状态特性更适合水平扩展与跨域交互，而 Session 适用于传统单体或共享存储架构。掌握二者本质是构建安全 API、设计 OAuth2/OIDC 协议及理解 AI Agent 身份鉴权的基础。

### 2. 底层原理剖析
1. Session/Cookie 机制：
   - 步骤：客户端发起请求 -> 服务端创建 Session 对象（内存/DB/Redis），生成唯一 Session ID -> 服务端返回 Set-Cookie: SessionId=<SID> -> 后续请求自动携带 Cookie Header -> 服务端解析 SID 查找 Session 数据。
   - 本质：状态转移于服务端，Cookie 仅是引用指针。依赖 HTTP Header 的透明传输特性。
   - 前端对比：类似 Java Web 中的 HttpSession，但 TS/JS 无法直接操作服务器内存，仅能通过浏览器 cookieStore 或 fetch headers 间接交互。

2. Token (JWT) 机制：
   - 结构：Header.Base64url(Algorithm, Type).Payload.Base64url(Claims).Signature.HMACSHA256(base64url(header)+"."+base64url(payload), secret_key)
   - 验证流程：客户端 Authorization: Bearer <Token> -> 服务端提取 Token -> 校验签名有效性（确保未篡改）-> 解析 Payload 提取 Claims -> 业务逻辑授权决策。
   - 本质：密码学签名的数据结构。服务端仅做验证，不做查询。具备自解释性（Self-describing）。
   - 关键差异：Session 状态更新需写操作（性能开销），Token 更新需重新签发（安全性高）。Session 易受 CSRF（同源策略保护除外）影响，Token 通常依赖 HTTPS 防止窃听。
   - 异同点：接口（Interface）定义契约，Session/Token 是实现该契约的不同载体。TS 接口描述形状，JWT Claims 描述事实，语法形式相似但语义不同（类型系统 vs 信任锚点）。

### 3. 基础代码与实战验证
```text
// 伪代码演示 JWT 核心验证逻辑 (Node.js/Vanilla JS 风格)
const crypto = require('crypto');

function verifyJWT(token, secretKey) {
  const parts = token.split('.');
  if (parts.length !== 3) throw new Error('Invalid format');

  const [headerB64, payloadB64, signatureB64] = parts;
  
  // 1. 重新计算签名以验证完整性
  const expectedSignature = base64UrlEncode(
    crypto.createHmac('sha256', secretKey)
      .update(`${headerB64}.${payloadB64}`)
      .digest()
  );

  // 2. 时序恒定比较防止侧信道攻击
  if (!crypto.timingSafeEqual(
    Buffer.from(signatureB64.replace(/-/g, '+').replace(/_/g, '/'), 'base64url'),
    Buffer.from(expectedSignature, 'base64') // 注意编码转换细节
  )) {
    throw new Error('Signature verification failed');
  }

  // 3. 解析载荷并检查过期时间 (exp) 等标准字段
  const payload = JSON.parse(Buffer.from(payloadB64.replace(/-/g, '+').replace(/_/g, '/'), 'base64url').toString());
  
  if (payload.exp && payload.exp < Date.now() / 1000) {
    throw new Error('Token expired');
  }

  return { header: JSON.parse(Buffer.from(headerB64, 'base64url').toString()), payload };
}
// 注释：此代码剥离了框架封装，直击 HMAC-SHA256 签名验证与 Base64URL 解码的本质过程。
```

### 4. 常见误区与进阶思考
误区 1：认为 JWT 可以保密存储任意敏感信息。JWT Payload 仅经过 Base64 编码而非加密，任何获取到 Token 的人均可解码读取内容（除非使用 JWE 加密）。严禁在其中存储密码、PII（个人身份信息）等高敏感数据，应仅存储非敏感的 Subject ID 和角色声明。
误区 2：混淆 Refresh Token 与 Access Token 的生命周期职责。Access Token 短命用于高频鉴权，Refresh Token 长寿用于换取新令牌。若将长有效期 Token 用于每次请求且缺乏旋转机制（Rotation），一旦泄露将面临持久化风险。
深度思考题：在分布式系统中，若要实现‘单点登出’（Single Sign-Out），即用户在一个地方注销后所有节点立即失效，由于 JWT 的无状态特性服务端无法主动撤销，请结合 Redis 黑名单机制或短寿命 Access Token + 异步刷新机制，设计一种兼顾性能与安全性的解决方案？
