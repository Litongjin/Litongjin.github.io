---
title: "每日基础技术总结 · 2026-09-15 · AES-GCM 的 nonce 复用与 GHASH 碰撞"
date: 2026-09-15 08:00:00
categories: [技术分享]
tags: ["技术分享", "安全基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-15 · AES-GCM 的 nonce 复用与 GHASH 碰撞

## 📚 今日主题

> **AES-GCM 的 nonce 复用与 GHASH 碰撞**（安全基础）

### 1. 核心概念速览
AES-GCM（Galois/Counter Mode）是 AEAD（带关联数据的认证加密）模式，由 AES-CTR 提供机密性，GHASH 提供完整性。nonce（IV）是 CTR 计数器基 inc32(J0) 与认证密钥派生输入 J0 的种子。GCM 的核心安全前提：同一密钥 K 下，任意 (K, nonce) 对至多使用一次。

本质：nonce 复用 => J0 相同 => CTR keystream 相同、E_K(J0) 相同；同时 GHASH 密钥 H=E_K(0^128) 固定不变。两个消息的标签异或会消去 E_K(J0)，得到 GHASH_H(M1) XOR GHASH_H(M2)。GHASH_H 是 GF(2^128) 上关于 H 的多项式求值，因此该异或给出一个以 H 为根的多项式方程；攻击者可求根恢复 H。恢复 H 后，GHASH 在固定 H 下是 GF(2) 线性映射，可构造碰撞或直接伪造任意消息标签。

位置：对称密码中的 AEAD 原语，广泛用于 TLS 1.3、IPsec、JWE、WebCrypto、磁盘加密、数据库字段加密。专业工程师必须掌握：AES-GCM 的 API 通常只接收 iv 字节串，不强制唯一性；nonce 管理是运行时密码学不变量，违反它会导致机密性与完整性同时崩溃。

### 2. 底层原理剖析
底层机制：
K: 128/192/256-bit AES 密钥；N: nonce；A: AAD；P: 明文。
H = E_K(0^128)
若 |N|=96 bit：J0 = N || 0^31 || 1
否则：J0 = GHASH_H(N || 0^s || [len(N)]_64)，其中 s 为填充到 128-bit 块边界的零比特数。
C = P XOR MSB_{len(P)}(GCTR_K(inc32(J0)))
S = GHASH_H(A || 0^v || C || 0^u || [len(A)]_64 || [len(C)]_64)
T = MSB_t(S XOR E_K(J0))

GHASH 递推（每个 128-bit 块 M_i，最后一块补齐）：
X_0 = 0
X_i = (X_{i-1} XOR M_i) • H
GHASH_H(M) = X_n = Σ_{i=1}^n M_i • H^{n-i+1}
其中 • 是 GF(2^128) 乘法，模 x^128 + x^7 + x^2 + x + 1。

nonce 复用攻击链：
1) 同 K 同 N => 同 J0 => 同 E_K(J0)，同 inc32(J0) => 同 CTR keystream KS。
   C1 XOR C2 = (P1 XOR KS) XOR (P2 XOR KS) = P1 XOR P2。
   若已知 P1，可直接恢复 P2；否则泄露明文差分。
2) 同 N 时 T1 XOR T2 = (GHASH_H(M1) XOR E_K(J0)) XOR (GHASH_H(M2) XOR E_K(J0)) = GHASH_H(M1) XOR GHASH_H(M2)。
   令差分块 D_i = M1_i XOR M2_i，则 GHASH_H(M1) XOR GHASH_H(M2) = Σ_{i=1}^n D_i • H^{n-i+1} = f(H)。
   攻击者获得 f(H)=T1 XOR T2，可在 GF(2^128) 上求根，恢复 H。
3) 恢复 H 后，利用 GHASH 的线性：GHASH_H(M XOR D) = GHASH_H(M) XOR GHASH_H(D)。
   选取非零 D 使 GHASH_H(D)=0，则 M 与 M XOR D 产生相同 GHASH，即 GHASH 碰撞；结合已知 E_K(J0)=T XOR GHASH_H(A,C)，可伪造任意消息标签。

与前端概念对比：WebCrypto 的 AES-GCM 参数 iv 类似 CSP nonce 或一次性 CSRF token，但破坏后果不是重放或脚本白名单绕过，而是直接暴露 GF(2^128) 上的认证密钥 H。更贴近 TypeScript 接口的编译期擦除：TS 接口只在编译期约束结构，运行时无强制；GCM 的 nonce 唯一性也不在 API 或类型系统内强制，它是调用方必须维护的运行时契约。Java 接口有运行时类型信息与分派，而 TS 接口没有；类似地，某些加密库可能在运行时检查 IV 长度，但绝不检查 IV 是否曾用过。唯一性只能由协议层持久化计数器或安全随机源保证。

### 3. 基础代码与实战验证
```text
const crypto = require('crypto');

const key = Buffer.alloc(16, 1); // 固定 128-bit AES 密钥
const iv = Buffer.alloc(12, 2);  // 固定 96-bit nonce：生产中绝不可复用
const p1 = Buffer.from('attack at dawn'); // 14 字节
const p2 = Buffer.from('defend at dusk'); // 14 字节

function gcmEncrypt(p) {
  const cipher = crypto.createCipheriv('aes-128-gcm', key, iv);
  const c = Buffer.concat([cipher.update(p), cipher.final()]);
  const tag = cipher.getAuthTag();
  return { c, tag };
}

const xor = (a, b) => Buffer.from(a.map((v, i) => v ^ b[i]));
const r1 = gcmEncrypt(p1);
const r2 = gcmEncrypt(p2);

console.log('C1 xor C2 =', xor(r1.c, r2.c).toString('hex'));
console.log('P1 xor P2 =', xor(p1, p2).toString('hex'));
// 二者相等：同 key 同 nonce 时 CTR 的 keystream 完全相同，密文异或直接约掉 keystream。

console.log('T1 xor T2 =', xor(r1.tag, r2.tag).toString('hex'));
// T1 xor T2 = GHASH_H(M1) xor GHASH_H(M2)。这是 H 的多项式在 H 处的值；
// 收集足够多同 nonce 消息后，可在 GF(2^128) 上求根恢复 H，再伪造标签。

// 恢复 H 的伪代码（不依赖框架，展示数学步骤）：
// 1. 将同 nonce 下两条消息的 AAD、密文、长度块按 GHASH 规则拼成 128-bit 块序列 M1_i、M2_i。
// 2. 计算差分块 D_i = M1_i XOR M2_i，构造多项式 f(x)=Σ D_i * x^{n-i+1}（GF(2^128) 乘法，模 x^128+x^7+x^2+x+1）。
// 3. 令 f(H)=T1 XOR T2，在 GF(2^128) 上求 f 的根；非零 f 至多有 n 个根，攻击者可通过求根恢复 H。
// 4. 用 H 计算 GHASH_H(A,C)，由 E_K(J0)=T XOR GHASH_H(A,C) 恢复 E_K(J0)。
// 5. 对任意伪造消息计算 S'=GHASH_H(A',C')，标签 T'=S' XOR E_K(J0)，通过验证。
```

### 4. 常见误区与进阶思考
误区 1：认为 nonce 复用只影响机密性，完整性仍由 128-bit tag 保护。实际：同 nonce 下 T1 XOR T2 消去 E_K(J0)，得到 GHASH_H 的差分多项式 f(H)。GHASH 不是抗碰撞哈希，而是固定 H 下的 GF(2^128) 线性多项式；攻击者可求根恢复 H，进而构造 GHASH 碰撞并伪造任意标签。机密性与完整性同时崩溃。

误区 2：认为“随机 96-bit nonce”天然安全。随机 nonce 存在生日碰撞界；NIST 对随机 IV 有使用次数限制，且多设备/多进程共享同一密钥时计数器状态若未持久化，重启、回滚、并发都会导致 nonce 重复。安全做法：同一密钥下使用确定性 96-bit 计数器 nonce，或 64-bit 随机前缀 + 32-bit 计数器，并保证崩溃恢复不重置计数器；更稳妥是每会话/每消息派生新密钥。

进阶思考题：同一 nonce 下加密两条消息，攻击者只知道两条密文与两个 tag，不知道任何明文。为什么此时不能直接恢复 H？需要补充什么条件才能把 T1 XOR T2 转化为关于 H 的可解多项式？请从 GHASH 输入块的构造与 GF(2^128) 多项式求根两个角度回答。
