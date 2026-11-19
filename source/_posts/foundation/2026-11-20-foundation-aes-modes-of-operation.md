---
title: "每日基础技术总结 · 2026-11-20 · 对称加密 AES 工作模式"
date: 2026-11-20 07:01:42
categories: [技术分享]
tags: ["技术分享", "安全进阶（Web / 认证授权 / 密码学）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-20 · 对称加密 AES 工作模式

## 📚 今日主题

> **对称加密 AES 工作模式**（安全进阶（Web / 认证授权 / 密码学））

### 1. 核心概念速览
核心概念速览：对称加密 AES 是分组密码，定义 128-bit 固定块上的可逆置换 E_k 与 D_k。AES 工作模式是围绕该置换构造的消息变换协议，把任意长度明文按固定块、反馈、计数器或认证加密策略映射成密文。它解决的问题不是 AES 本身是否安全，而是如何让一个固定块置换变成工程可执行、语义安全、可并行、可流式、可认证、可失败处理的加密协议。ECB 暴露明文结构，CBC 需要不可预测 IV 与 padding，CTR 将块密码转成密钥流，GCM 在 CTR 上叠加 GHASH 认证。它位于 TLS、JWE、数据库透明加密、对象存储、模型权重保护、安全推理、联邦学习等体系的基础层。专业工程师必须掌握，因为模式误用会直接导致密钥复用、明文结构泄露、密文篡改不被发现、性能退化与协议兼容故障。

### 2. 底层原理剖析
底层原理剖析：
1. AES 核心是置换：明文块 P_i 与密文块 C_i 都是 16 字节。加密函数 E_k 与解密函数 D_k 互为逆运算。工作模式决定如何重复调用 E_k/D_k，以及状态是否跨越分组传递。
2. ECB：C_i = E_k(P_i)。每块独立，无 IV，同明文块产生同密文块，破坏语义安全并暴露重复结构；错误传播独立，可并行；除教学或极少固定短字段外不应用于通用机密性。
3. CBC：C_i = E_k(P_i xor C_{i-1})，IV 充当 C_0。解密 P_i = D_k(C_i) xor C_{i-1}。需要唯一且不可预测 IV 才能获得语义安全；需要 PKCS#7 类 padding；加密串行，解密可并行；一个密文块被篡改会污染当前明文块和下一明文块的解密边界。
4. CTR：把 AES 转成流密码。T_i = E_k(nonce || counter_i)，C_i = P_i xor T_i。无需 padding，加解密都用加密方向，可并行；但同一密钥下 nonce || counter 序列绝不复用，复用会直接产生可 XOR 的明文差值。
5. GCM：AEAD 模式。内部用 CTR 加密明文/密文，再用 GHASH 对 AAD、密文与长度信息生成认证 tag。标准 96-bit nonce 下 J0 = nonce || 0^31 || 1，CTR counter 从 inc32(J0) 开始，Tag = GHASH(AAD, CT, H) xor AES-Enc(J0)。Tag 必须验证后才能释放明文；GCM 并行性好，是现代传输与认证加密存储的主流，但 nonce 唯一性是硬约束。
6. CFB/OFB：反馈式流密码。CFB 将加密输出与密文/反馈组合，OFB 将密钥流自身反馈。它们可处理变长数据且解密可用加密方向，但状态串行、无 AEAD，随机化与性能不如 CTR/GCM。
7. 与前端体系类比：JS 中 crypto.subtle.encrypt 的 algorithm.name 只是选择运行时策略，不能把接口可用误认为语义安全。Java 接口是名义类型契约，TS 接口是结构类型编译期约束；AES 模式不是类型形状，而是状态机与协议约束。ECB/CBC/CTR/GCM 的关系更像 HTTP method 与状态转移：调用签名可能相似，但副作用、幂等、并发、认证边界完全不同。前端工程师需要像审查 API 幂等、状态转移和失败路径一样审查 IV/nonce、padding、tag 与认证失败。

### 3. 基础代码与实战验证
```text
基础代码与实战验证：
// Node.js 内置 crypto 验证 AES-CBC 与 AES-GCM 的基础行为；无第三方框架。
const crypto = require('crypto');

// CBC: 16 字节密钥对应 AES-128-CBC；IV 必须是唯一且不可预测的初始链接块。
const key = crypto.randomBytes(16);
const iv = crypto.randomBytes(16);
const encrypt = crypto.createCipheriv('aes-128-cbc', key, iv);
encrypt.setAutoPadding(true); // 自动 PKCS#7 padding，补齐到 16 字节块。
const plaintext = Buffer.from('AES mode matters');
const ciphertext = Buffer.concat([encrypt.update(plaintext), encrypt.final()]);

const decrypt = crypto.createDecipheriv('aes-128-cbc', key, iv);
const recovered = Buffer.concat([decrypt.update(ciphertext), decrypt.final()]);

// 篡改 ciphertext[0] 后，CBC 解密可能因 padding/完整性检查失败；GCM 可显式认证标签。
try { const bad = Buffer.from(ciphertext); bad[0] ^= 0x01; decrypt.update(bad); decrypt.final(); } catch (e) {}

// GCM: 使用 12 字节 nonce；同一密钥下 nonce 复用会破坏机密性，必须工程约束。
const gcmKey = crypto.randomBytes(32);
const nonce = crypto.randomBytes(12);
const gcmEncrypt = crypto.createCipheriv('aes-256-gcm', gcmKey, nonce);
const gcmCiphertext = Buffer.concat([gcmEncrypt.update(plaintext), gcmEncrypt.final()]);
const authTag = gcmEncrypt.getAuthTag(); // GHASH over nonce/AAD/ciphertext + final block.

const gcmDecrypt = crypto.createDecipheriv('aes-256-gcm', gcmKey, nonce);
gcmDecrypt.setAuthTag(authTag);
const recoveredGcm = Buffer.concat([gcmDecrypt.update(gcmCiphertext), gcmDecrypt.final()]);

// 篡改 GCM 密文或 nonce 后，final 应因认证失败抛错；这是 AEAD 的核心语义。
try { const tamper = Buffer.from(gcmCiphertext); tamper[0] ^= 0x01; gcmDecrypt.update(tamper); gcmDecrypt.final(); } catch (e) {}

// 模式本质伪代码（16 字节块）：
// CBC: C0 = IV; C[i] = AES-Enc(P[i] xor C[i-1]); P[i] = AES-Dec(C[i]) xor C[i-1].
// CTR: T[i] = AES-Enc(nonce || counter + i); C[i] = P[i] xor T[i].
// GCM: H = AES-Enc(0); J0 = nonce || 0^31 || 1; CT via CTR counters; Tag = GHASH(AAD, CT, H) xor AES-Enc(J0).
// 关键约束：CBC IV 不可预测；CTR/GCM nonce 绝不复用；GCM 必须验证 Tag 才能使用明文。
```

### 4. 常见误区与进阶思考
常见误区与进阶思考：
1. 常见误区：把 AES 当成“直接加密字符串”的黑盒，把 IV 与 nonce 混为一谈。CBC IV 必须每次不可预测且不复用，但它不等于 GCM nonce；GCM nonce 一旦在同一密钥下复用，会同时破坏机密性与认证可靠性，因为同一密钥流与同一 GHASH 约束被重复使用，攻击者可 XOR 密文恢复明文差值，或构造重放与伪造。ECB 即便实现最简单，也只是同明文块映射同密文块，不能用于变长敏感数据。另一个误区是只验证“能解密”而不验证认证标签或 padding 失败路径，CBC/CTR 这类非 AEAD 模式天然缺乏完整性边界。
2. 进阶思考题：如果系统采用 AES-256-GCM，随机生成 96-bit nonce，每条请求一条消息，密钥 7 天轮换一次，日均 1e10 条消息。请按生日界估算 nonce 碰撞概率与风险，并解释为什么“随机 nonce 足够长”仍不足以成为工程安全结论；还需要哪些约束，例如唯一生成器、计数器、域分离、重试幂等、失败不重用 nonce，以及认证标签必须绑定消息长度、协议版本、密钥 ID 与上下文字段。
