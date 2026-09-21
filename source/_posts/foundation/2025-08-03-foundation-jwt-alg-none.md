---
title: "每日基础技术总结 · 2025-08-03 · JWT：签名验证、alg:none 攻击与刷新"
date: 2025-08-03 20:00:00
categories: [技术分享]
tags: ["技术分享", "安全进阶（Web / 认证授权 / 密码学）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2025-08-03 · JWT：签名验证、alg:none 攻击与刷新

## 📚 今日主题

> **JWT：签名验证、alg:none 攻击与刷新**（安全进阶（Web / 认证授权 / 密码学））

### 1. 核心概念速览
JWT (JSON Web Token) 是一种开放标准 RFC 7519，用于在各方之间作为 JSON 对象安全地传输信息。其本质是自包含（Self-contained）的无状态认证令牌，由 Header、Payload 和 Signature 三部分组成，中间以 '.' 分隔。它解决的核心问题是分布式系统中的身份声明传递与完整性校验。机制上通过非对称或对称加密算法对头部和载荷进行数字签名，接收方仅验证签名即可确认数据未被篡改且来源可信。对于全栈工程师而言，掌握 JWT 是理解从 Session 集中式管理转向微服务/Serverless 无状态架构的关键桥梁，也是构建现代 Web API 安全体系的基石。

### 2. 底层原理剖析
JWT 的结构遵循 JWS (JSON Web Signature) 规范。Header 定义算法 (alg) 和类型 (typ)，Payload 承载 Claim（声明），Signature 是对 Header.b64url(Header) + '.' + b64url(Payload) 使用指定密钥和 alg 算法计算得出的摘要。解析流程为：1. 分割字符串获取 Head 和 Payload；2. Base64Url 解码还原二进制数据并反序列化 JSON；3. 根据 alg 字段选择验证器；4. 使用共享密钥或公钥重新计算签名并与 Token 中的签名对比。关键机制在于 'b64url' 编码避免了传统 Base64 中的 '+' 和 '/' 字符导致的 URL 安全问题。对比前端概念：TS 接口定义类型契约（编译时检查），JWT Header/Payload 则是运行时的数据契约（运行时校验）。JWT 类似于 HTTP Header 中的 Authorization 字段，但将上下文信息嵌入 Token 本身而非依赖服务端会话存储。

### 3. 基础代码与实战验证
```text
import jwt from 'jsonwebtoken'; // 假设使用 node-jwt 库，底层处理 HMAC/RSAA
const SECRET_KEY = process.env.JWT_SECRET;

// 1. 签发 Token (Sign)
const createToken = () => {
    const payload = { userId: 101, role: 'admin', exp: Math.floor(Date.now()/1000) + 3600 };
    // header 默认 {alg: 'HS256', typ: 'JWT'}
    return jwt.sign(payload, SECRET_KEY); 
};

// 2. 攻击演示: alg:none 漏洞原理
const attackToken = () => {
    // 构造恶意 Header: {"alg":"none","typ":"JWT"}
    // 伪造 Payload: {"userId":1,"role":"super_admin"}
    // 签名部分留空
    const part1 = Buffer.from(JSON.stringify({alg:'none',typ:'JWT'})).toString('base64url');
    const part2 = Buffer.from(JSON.stringify({userId:1,role:'super_admin'})).toString('base64url');
    return `${part1}.${part2}.`;
};

// 3. 验证逻辑 (伪代码核心)
const verifyToken = (token) => {
    try {
        // 正规库应强制要求 alg 匹配预期列表，阻止 none
        const decoded = jwt.verify(token, SECRET_KEY);
        return decoded; // 返回 Payload
    } catch (e) {
        throw new Error('Invalid Token');
    }
};

// 4. 刷新机制 (Refresh Pattern)
const refreshFlow = () => {
    // Access Token 短寿命 (如 15min), Refresh Token 长寿命 (如 7d)
    // RT 通常存储在 HttpOnly Cookie 中以防 XSS
    const newAT = jwt.sign(newPayload, AT_SECRET, { expiresIn: '15m' });
    // ST 验证后发新 AT，不修改 RT 本身或轮换 RT Key
    return { accessToken: newAT };
};
```

### 4. 常见误区与进阶思考
1. alg:none 攻击与算法混淆：若服务端未严格校验 Header 中的 alg 字段，攻击者可将其篡改为 'none' 并去除签名，导致服务端跳过验签直接信任 Payload。此外，HMAC (HS256) 与 RSA (RS256) 混用风险：若公钥泄露却用了 HS256 验证，攻击者可用公钥当作私钥生成签名。必须配置 allowlist 强制指定预期的算法。
2. 刷新策略的认知偏差：许多开发者误以为旋转 Refresh Token 即绝对安全。实际上，RT 存储在 HttpOnly Cookie 中仍需防范 CSRF（需 SameSite=Strict/Lax + CSRF Token）以及 XST (Cross-Site Tracing) 等边缘侧威胁。同时，Access Token 一旦发出不可撤销（除非引入黑名单机制增加复杂度），因此短时效设计是权衡安全与性能的唯一解。思考题：在无数据库轮询黑名单的情况下，如何利用 JTI (JWT ID) 配合内存缓存实现短生命周期的访问控制撤销？
