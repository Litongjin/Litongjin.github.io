---
title: "每日基础技术总结 · 2026-09-12 · JWT 签名验证中的算法混淆攻击 (Algorithm Confusion) 原理与防御"
date: 2026-09-12 08:00:00
categories: [技术分享]
tags: ["技术分享", "安全基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-12 · JWT 签名验证中的算法混淆攻击 (Algorithm Confusion) 原理与防御

## 📚 今日主题

> **JWT 签名验证中的算法混淆攻击 (Algorithm Confusion) 原理与防御**（安全基础）

### 1. 核心概念速览
算法混淆攻击（Algorithm Confusion）是针对 JWT 签名验证的一种类型混淆攻击。攻击者篡改 JWT 头部的 alg 字段，使其从非对称算法（如 RS256）切换为对称算法（如 HS256），诱导验证方使用同一把公钥（公开已知）作为 HMAC 对称密钥来验签，从而伪造任意有效令牌。本质是验证方对来自不可信输入的算法标识缺少白名单约束，且未对密钥材料与算法类型进行一致性校验。该问题属于 JWT 实现安全中最经典的高危配置错误，破坏认证的完整性与来源不可否认性。在全栈与 AI 后端开发中，掌握此机制是建立安全边界、设计高信度认证链路的基本功；它与前端 TypeScript 的编译期类型约束、Java 接口的强制实现契约形成对照——缺少显式运行时约束时，动态输入可绕过静态假设。

### 2. 底层原理剖析
JWT 结构为 header.payload.signature。正常情况下 RS256 验签：验证方持有公钥，对 header.payload 做 RSA 签名验证。攻击流程如下：
1. 攻击者拿到合法 JWT，读取其公钥（公开）。
2. 将 header 中的 alg 改为 HS256，payload 改为任意授权数据。
3. 使用公钥字符串（PEM/DER）作为 HMAC 密钥，对 header.payload 计算 HMAC-SHA256 签名。
4. 将伪造 token 发送给验证方。
5. 验证方解析 header，发现 alg=HS256；由于代码未固定算法白名单，便调用 HMAC 验签函数，并将配置中读取的公钥作为对称密钥。
6. 因为攻击者签名时用了完全相同的密钥材料，HMAC 校验通过，token 被接受。

伪代码：
verify(token, key) {
  header = b64decode(token.part[0])
  if (header.alg == 'RS256') return rsa_verify(key, token)
  if (header.alg == 'HS256') return hmac_verify(key, token)
}

漏洞根因：算法选择来自不可信输入，且没有将 key 的类型与算法进行匹配。防御原则：验证端固定允许的算法白名单，并确保非对称算法使用公钥对象，对称算法使用独立管理的共享密钥。

与前端工程概念对比：TypeScript 的接口是静态结构类型，编译后无运行时信息；Java 的接口是编译期强制实现的契约。JWT 的 alg 字段则像一个未受运行时校验的 '接口名'，验证方若直接信任它，就等同于将类型转换的决策权交给攻击者——动态语言中的 '鸭子类型' 被滥用。

### 3. 基础代码与实战验证
```text
演示攻击与防御的极简 Node.js 脚本核心片段（基于 jsonwebtoken 库，避免使用任何框架）：

攻击者视角：
const fs = require('fs');
const jwt = require('jsonwebtoken');
const publicKey = fs.readFileSync('public.pem', 'utf8'); // 公钥是公开的，可作为 HMAC 密钥
const forged = jwt.sign({ user: 'admin', role: 'admin' }, publicKey, { algorithm: 'HS256' });
// 注意：这里生成的 token 的 header.alg 为 HS256，签名使用公钥字符串作为 HMAC 密钥。

若验证方代码存在漏洞：
jwt.verify(forged, publicKey, { algorithms: ['HS256', 'RS256'] });
// 漏洞在于：algorithms 数组允许 HS256，且公钥被直接当作对称密钥传入。
// 验证流程：解析 header 得 alg=HS256，然后执行 HMAC 校验，key=publicKey。攻击者的签名正是用同样的 key 计算，因此通过。

防御要点（正确代码）：
jwt.verify(forged, publicKey, { algorithms: ['RS256'] });
// 显式白名单只允许 RS256，HS256 请求直接拒绝，算法混淆失效。

若不使用库，原理伪代码如下：
function unsafeVerify(token, key) {
  const [headerB64, payloadB64, sigB64] = token.split('.');
  const header = JSON.parse(base64url(headerB64));
  const input = headerB64 + '.' + payloadB64;
  if (header.alg === 'HS256') {
    return HMAC_SHA256(key, input) === sigB64; // key 是公钥字符串
  }
  if (header.alg === 'RS256') {
    return RSA_Verify(key, input, sigB64); // key 是公钥对象
  }
  return false;
}
// 该函数未固定算法，也未根据算法校验 key 类型，正是漏洞。
```

### 4. 常见误区与进阶思考
误区1：认为只有 HS256 才能被用来混淆。实际只要验证方允许多种算法，并且密钥材料可以被攻击者获取或推导，就可能发生算法混淆，例如 RS256 与 HS256、ES256 与 HS256 等。
误区2：认为密钥强度足够就能防御。算法混淆攻击的关键不是密钥强度，而是验证逻辑信任了 token 头部的 alg 字段；即便使用强密钥，公钥被公开就足以伪造对称签名。
思考题：若验证方同时支持 RS256 和 HS256，且分别管理 RSA 公钥和 HMAC 共享密钥，攻击者将 alg 改为 HS256 后又如何命中 HMAC 密钥？防御时能否在密钥管理层面为每个密钥绑定允许的算法，从而彻底消除算法混淆？请从验签函数参数传递链的角度给出你的方案。
