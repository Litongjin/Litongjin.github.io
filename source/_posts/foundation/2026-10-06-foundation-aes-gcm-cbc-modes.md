---
title: "每日基础技术总结 · 2026-10-06 · 对称加密 AES 工作模式（GCM/CBC）"
date: 2026-10-06 07:05:38
categories: [技术分享]
tags: ["技术分享", "安全进阶（Web / 认证授权 / 密码学）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-06 · 对称加密 AES 工作模式（GCM/CBC）

## 📚 今日主题

> **对称加密 AES 工作模式（GCM/CBC）**（安全进阶（Web / 认证授权 / 密码学））

### 1. 核心概念速览
AES 工作模式是分组密码算法在真实数据流上的操作范式，决定如何将明文块与密钥、初始化向量组合成密文。AES 本身是固定 128 位块、128/192/256 位密钥的 Feistel-Like SPN 结构，而 CBC/GCM 等模式定义其加密语义与安全属性。CBC 通过链式 XOR 与 IV 实现语义安全，但缺乏完整性；GCM 在 CTR 模式基础上引入 GHASH 认证标签，提供 AEAD 认证加密。这是密码学工程化的最小必要单元，也是 TLS、JWT 加密、文件加密、Web Crypto API 的核心抽象。前端工程师常接触 WebCrypto 与 HTTPS，但若不理解工作模式差异，无法判断何时该用 AES-GCM、何时禁用 ECB/CBC，也无法理解 nonce 重用、padding oracle、tag 伪造等攻击面。在 AI 系统中，模型权重加密、推理数据隔离、联邦学习通信安全均依赖正确选型。

### 2. 底层原理剖析
AES 是块密码，单块加密函数 E_K(P) -> C，P/C 为 128 位。工作模式解决三个问题：1. 明文长度 > 块长；2. 相同明文块不能产生相同密文块；3. 是否提供完整性。

CBC 模式：
C_i = E_K(P_i XOR C_{i-1}), C_0 = IV
解密：P_i = D_K(C_i) XOR C_{i-1}
特点：加密串行、解密可并行；需 PKCS#7 填充；无认证，易受 padding oracle 攻击；IV 必须不可预测。

GCM 模式：
1. 用 AES-CTR 加密：J0 = IV || 0^31 || 1（96-bit IV），后续 counter 递增；
2. 用 GHASH 计算认证标签：H = E_K(0^128)，对 AAD 与密文按 GF(2^128) 多项式求值；
3. Tag = GHASH(H, A, C) XOR E_K(J0)。
特点：无需填充、支持并行、提供认证加密、可附加 AAD；nonce 重用会泄露密钥流并破坏完整性。

对比前端概念：AES-GCM 与 AES-CBC 的关系类似 TypeScript 的 interface 与 Java 的 interface。TS interface 是结构化类型，仅约束形状，不保证运行时行为；Java interface 是名义类型且可携带默认方法。CBC 仅约束“加密”形状，GCM 则在加密之上绑定完整性语义，二者不可互换。

### 3. 基础代码与实战验证
```text
// Node.js crypto 极简验证 AES-256-GCM 与 AES-256-CBC
const crypto = require('crypto');

// 1. 生成密钥与随机 nonce/IV
const key = crypto.randomBytes(32); // AES-256 密钥 32 字节 = 256 bits
const iv = crypto.randomBytes(12);  // GCM 推荐 12 字节 nonce，形成 J0 = IV || 0x00000001
const plain = Buffer.from('secure-payload');

// 2. AES-256-GCM 加密：CTR 流加密 + GHASH 认证
const gcm = crypto.createCipheriv('aes-256-gcm', key, iv);
const gCipher = Buffer.concat([gcm.update(plain), gcm.final()]);
const gTag = gcm.getAuthTag(); // 16 字节认证标签，验证密文完整性与真实性

// 3. AES-256-GCM 解密：先验证 tag 再解密，防篡改
gcm.setAuthTag(gTag);
const gPlain = Buffer.concat([gcm.update(gCipher), gcm.final()]);

// 4. AES-256-CBC 加密：链式分组 + PKCS#7 填充，无完整性保护
const cbcIv = crypto.randomBytes(16); // CBC 块大小 16 字节，IV 必须随机不可预测
const cbc = crypto.createCipheriv('aes-256-cbc', key, cbcIv);
const cCipher = Buffer.concat([cbc.update(plain), cbc.final()]); // final 触发填充与尾块加密
```

### 4. 常见误区与进阶思考
误区一：认为 AES 密钥足够长就安全，忽略模式选择与 nonce 管理。CBC 无认证，攻击者可篡改密文并利用 padding oracle 恢复明文；GCM nonce 重用会导致 CTR 密钥流异或泄露与认证完全失效。
误区二：将 IV/nonce 当作秘密参数。IV/nonce 可公开，但必须唯一（GCM）或不可预测（CBC），否则破坏语义安全。

思考题：在 TLS 1.3 中，为何只保留 AES-GCM 与 ChaCha20-Poly1305，而淘汰 AES-CBC？从填充预言、延迟攻击、认证加密语义三个角度分析其必然性。
