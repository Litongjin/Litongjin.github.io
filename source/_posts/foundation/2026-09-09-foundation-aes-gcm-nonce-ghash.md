---
title: "每日基础技术总结 · 2026-09-09 · AES-GCM 的 nonce 复用与 GHASH 碰撞"
date: 2026-09-09 08:00:00
categories: [技术分享]
tags: ["技术分享", "安全基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-09 · AES-GCM 的 nonce 复用与 GHASH 碰撞

## 📚 今日主题

> **AES-GCM 的 nonce 复用与 GHASH 碰撞**（安全基础）

### 1. 核心概念速览
AES-GCM 是一种 AEAD，由 CTR 模式加密和 GHASH 通用散列两部分协同工作。nonce 的唯一性是整个安全模型的硬前提：nonce 决定 CTR 的起始计数块从而决定密钥流，也决定 J0 从而决定 tag 掩码 E_K(J0)。nonce 复用意味着两条消息的密钥流完全相同、tag 掩码完全相同：密文异或直接等于明文异或；tag 异或变成长度为密文块数的、系数完全已知的 GF(2^128) 多项式在 GHASH 密钥 H 处的取值。GHASH 本质是 H 上的多项式求值，是 ε-Δ-universal 哈希；没有 H 时碰撞概率约为 O(块数/2^128)，一旦 nonce 复用，H 可通过求根/开方解得，任意构造 GHASH 碰撞成为可能。该知识点属于对称密码学、认证加密和攻击模型的核心，专业工程师必须理解 nonce 唯一性是协议层约束，算法本身不强制也不校验；否则会在设计加密消息协议时留下灾难性漏洞。

### 2. 底层原理剖析
底层机制：设 H=E_K(0^128)。GHASH 把 AAD 和密文以及长度块拼接并补齐为 128-bit 块序列 X_1...X_t，计算 X_0=0，对每个块执行 X_i=(X_{i-1}⊕X_i)·H。展开得到 GHASH_H(X)=⊕_{i=1..t} X_i·H^{t-i+1}。最终 tag = GHASH_H(X)⊕E_K(J0)，其中 J0=nonce||0^31||1。若同一 nonce 加密两条消息，则 E_K(J0) 相同，于是 T_1⊕T_2 = GHASH_H(X_1)⊕GHASH_H(X_2) = ⊕ (X_1[i]⊕X_2[i])·H^{t-i+1}。这是以 H 为未知数的多项式方程，右侧所有系数都从密文/长度块推出。取无 AAD、长度 16 字节的两条消息，块序列只有 X=[C,L]，L=128；因为长度相同，L 项抵消，得到 T_1⊕T_2=(C_1⊕C_2)·H^2。GF(2^128) 特征为 2，平方映射是双射，所以 H=sqrt((T_1⊕T_2)/(C_1⊕C_2))，唯一解。恢复 H 后即可对任意密文块计算 GHASH_H，再用 E_K(J0)=T_1⊕GHASH_H(X_1) 恢复 tag 掩码，实现完全伪造。在 nonce 不复用时，攻击者面对随机 H；给定两个不同块的 X,Y，GHASH_H(X)⊕GHASH_H(Y) 是一个非零多项式在 H 处的取值，最多 t 次，故碰撞概率不超过 t/2^128。nonce 复用的本质是给攻击者提供了同一个 H 下的多个求值点，把高次碰撞问题降为单点求解。与前端概念的类比：Java interface 是运行时保留的 nominal 类型边界，TS interface 在编译期被擦除，不在运行时存在；GCM 的 nonce 唯一性也类似 TS 的编译期约定——算法内部不保存、不检查任何「是否已用过」的元数据。违反约定时，Java/TS 的差异只会导致类型问题，而 GCM 的违反会直接暴露密钥流和 H 的结构，安全性发生不可逆坍缩。本质上，它们都在提醒工程抽象的安全边界必须由调用方在运行时遵守，仅靠编译期或数学前的约定不足以防住强对抗场景。

### 3. 基础代码与实战验证
```text
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import os

# GF(2^128) 乘法，不可约多项式 x^128+x^7+x^2+x+1
def gf_mul(a, b):
    r = 0
    for _ in range(128):
        if b & 1:
            r ^= a
        b >>= 1
        if a >> 127:
            a = ((a << 1) & ((1 << 128) - 1)) ^ 0x87
        else:
            a = (a << 1) & ((1 << 128) - 1)
    return r

def gf_inv(x):
    # 费马小定理求逆：x^(2^128-2)
    res = 1
    e = (1 << 128) - 2
    while e:
        if e & 1:
            res = gf_mul(res, x)
        x = gf_mul(x, x)
        e >>= 1
    return res

def gf_sqrt(x):
    # 特征 2 的域上平方根：sqrt(x)=x^(2^127)
    res = 1
    e = 1 << 127
    while e:
        if e & 1:
            res = gf_mul(res, x)
        x = gf_mul(x, x)
        e >>= 1
    return res

def ghash_single(c_block, length_int):
    # 无 AAD 的单密文块 GHASH：((C*H) xor L)*H
    return gf_mul(gf_mul(c_block, H) ^ length_int, H)

# 生成密钥并故意用同一 nonce 加密两条 16 字节消息
key = os.urandom(16)
nonce = os.urandom(12)
gcm = AESGCM(key)
c1 = gcm.encrypt(nonce, b'attack at dawn --', None)
c2 = gcm.encrypt(nonce, b'defend at noon --', None)

C1 = int.from_bytes(c1[:-16], 'big')
C2 = int.from_bytes(c2[:-16], 'big')
T1 = int.from_bytes(c1[-16:], 'big')
T2 = int.from_bytes(c2[-16:], 'big')
L = 128  # 长度块：AAD位数(0)左移64位，与密文位数128合并后为128

# 攻击：T1 xor T2 = (C1 xor C2) * H^2，所以先求 H
delta_T = T1 ^ T2
delta_C = C1 ^ C2
H = gf_sqrt(gf_mul(delta_T, gf_inv(delta_C)))

# 利用第一条密文恢复 tag 掩码 E_K(J0)
mask = ghash_single(C1, L) ^ T1

# 构造任意 16 字节伪造密文并计算合法 tag
C_forged = 0x1234567890abcdef1234567890abcdef
T_forged = ghash_single(C_forged, L) ^ mask
print(C_forged.to_bytes(16, 'big'), T_forged.to_bytes(16, 'big'))
```

### 4. 常见误区与进阶思考
误区1：把 nonce 复用只当作机密性问题。实际上 nonce 复用同时摧毁认证性：两条消息 tag 的异或消去 E_K(J0)，剩下的方程让攻击者能解出 H，随后对任意密文构造合法 tag——这是完整的伪造攻击。误区2：以为随机 H 下 GHASH 碰撞概率天然很小就足够安全。该概率依赖 H 的均匀随机性，nonce 复用把均匀随机的 H 变成可计算出来的固定值，碰撞概率直接变成 1；因此任何要求唯一 nonce 的协议都不能用随机 nonce 的生日概率来豁免。思考题：在无 AAD、单块密文且未知明文的情况下，仅凭两条同 nonce 的 (C,T) 能否解出 H？若能，列出 GF(2^128) 上的代数步骤；若不能，指出缺失量，并解释这与「nonce 复用可伪造」是否矛盾。
