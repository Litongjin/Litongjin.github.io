---
title: "每日基础技术总结 · 2025-02-24 · JWT 结构、签名验证与常见攻击（none 算法）"
date: 2025-02-24 20:00:00
categories: [技术分享]
tags: ["技术分享", "Java 后端与 Spring 生态"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2025-02-24 · JWT 结构、签名验证与常见攻击（none 算法）

## 📚 今日主题

> **JWT 结构、签名验证与常见攻击（none 算法）**（Java 后端与 Spring 生态）

### 1. 核心概念速览
JSON Web Token (JWT) 是一种开放标准 RFC 7519，用于在各方之间作为 JSON 对象安全地传输信息。其本质是无状态的认证/授权机制，将身份信息编码后通过数字签名附加在 Token 中，服务端仅凭签名验证即可确认来源可信且未被篡改，无需查询数据库或会话存储。

结构：Header (算法/类型) . Payload (声明数据) . Signature (签名)。其中 Header 和 Payload 采用 Base64Url 编码（明文可见），仅 Signature 部分保障完整性与真实性。

为什么必须掌握：前端开发者常误以为 JWT 是加密数据，实则它是签名数据。混淆 '保密性' (Confidentiality) 与 '完整性' (Integrity) 会导致严重的安全架构缺陷。理解 JWT 机制是构建无状态微服务、OAuth2/OIDC 集成以及防御常见身份伪造攻击的基础。

### 2. 底层原理剖析
1. 生成流程：
   H = Base64UrlEncode({"alg": "HS256", "typ": "JWT"})
   P = Base64UrlEncode({"sub": "1234567890", "name": "John Doe", ...})
   RS = HMACSHA256(Base64UrlEncode(H) + "." + Base64UrlEncode(P), secret_key)
   T = H + "." + P + "." + RS

2. 验证流程：
   - 解析分割符 '.'，提取 Header (H_p) 和 Payload (P_p) 及 Signature (S_p)。
   - 使用约定算法重新计算 HMACSHA256(H_p + "." + P_p, secret_key)，得到 S_calc。
   - 恒定时间比较 (Constant-time Comparison) S_calc 与 S_p。

3. 与前端 TS 接口对比：
   - Java Interface vs TS Interface: Java 接口是运行时多态契约，编译期擦除但 JVM 反射可感知；TS 接口纯编译期存在，运行时无任何痕迹。
   - JWT Header vs TS Interface: JWT 的 Header 如同 TS 接口定义，指定了数据结构（Claim）和元数据（Alg），但它存在于运行时的明文载荷中，且可以被客户端任意修改。TS 接口约束前端代码结构，JWT Header 约束后端解析策略。若后端信任未经验证的 Header 中的 alg 字段（如允许 none 或 HS256 强转为 RS256），则等同于前端信任用户传入的任何 JSON 结构而不调用类型守卫。

4. None 算法攻击原理：
   - 漏洞点：某些库实现中，若 header.alg 设为 'none'，签名逻辑被跳过，返回空字符串。
   - 利用：构造 Header={'alg':'none','typ':'JWT'}，Payload={user: admin}，Signature=''。若服务器端配置允许 'none' 或从 JWT Header 读取算法时未做白名单限制，签名验证函数直接返回 true，导致无条件信任任意 Token。

### 3. 基础代码与实战验证
```text
// 模拟 JWT 解析与签名验证核心逻辑 (Java 风格伪代码)
public boolean validateToken(String token, String secretKey) {
    String[] parts = token.split("\\."); // 注意：split 需谨慎处理多个分隔符情况
    if (parts.length != 3) return false;
    
    String headerJson = new String(decode(parts[0])); // Base64Url 解码
    String payloadJson = new String(decode(parts[1]));
    byte[] signatureBytes = decode(parts[2]);
    
    // 关键步骤1：解析 Header 以获取预期算法
    JwtHeader header = parse(headerJson);
    
    // 【安全关键】禁止直接使用 header.alg 作为验签算法
    // 错误做法：Algorithm algorithm = Algorithm.valueOf(header.getAlgorithm());
    // 正确做法：强制限定为对称密钥算法，除非你有明确的非对称公钥交换机制
    String expectedAlg = "HS256"; 
    
    // 防止 none 攻击：如果 header 声称是 none，直接拒绝
    if ("none".equalsIgnoreCase(expectedAlg)) { 
        throw new SecurityException("Algorithm none is not allowed"); 
    }

    // 关键步骤2：重新计算签名
    String inputToSign = parts[0] + "." + parts[1];
    Mac mac = Mac.getInstance("HmacSHA256");
    mac.init(new SecretKeySpec(secretKey.getBytes(), "HmacSHA256"));
    byte[] calculatedSignature = mac.doFinal(inputToSign.getBytes(UTF_8));
    
    // 关键步骤3：恒定时间比较，防止时序侧信道攻击
    return MessageDigest.isEqual(calculatedSignature, signatureBytes);
}
```

### 4. 常见误区与进阶思考
1. 认知误区：认为 'Base64 编码' 等于 '加密'。
   - 真相：Base64 是可逆的编码格式，任何人皆可解码查看 Payload 内容。JWT 不保证保密性。敏感信息（密码、PII）绝不应放入 JWT Payload。如需保密，应配合 TLS 或在 Application Layer 额外加密 (JWE)。

2. 认知误区：信任 Header 中的 'alg' 字段。
   - 真相：Header 也是 Base64 编码，客户端可随意篡改。攻击者可将 alg 改为 'none' 绕过签名，或利用 'key confusion' 攻击（例如将 RSA 公钥伪装成 HMAC 密钥进行 HS256 签名）。服务端必须硬编码期望的算法集合，严禁动态信任客户端传入的算法描述。

思考题：在设计支持 RS256 (非对称) 的 JWT 系统时，如果攻击者截获了你的公钥 (Public Key)，他能否伪造一个让服务端验签通过的恶意 Token？如果不能，请阐述公钥分发的局限性；如果能，请说明此时 JWT 的哪种属性被破坏了，以及应引入什么机制（如 JWKS 地址 + Key ID Kid 校验）来缓解。
