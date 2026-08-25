---
title: "每日基础技术总结 · 2026-08-25 · AES-GCM 的 nonce 复用与 GHASH 碰撞"
date: 2026-08-25 08:00:00
categories: [技术分享]
tags: ["技术分享", "安全基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-25 · AES-GCM 的 nonce 复用与 GHASH 碰撞

## 📚 今日主题

> **AES-GCM 的 nonce 复用与 GHASH 碰撞**（安全基础）

### 1. 核心概念速览
核心概念速览：AES-GCM 是一种 AEAD 结构，由 AES-CTR 提供机密性、GHASH 提供完整性认证。GHASH 是一个定义在 GF(2^128) 上的多项式评估函数，认证标签 T = E_K(J0) XOR GHASH(H, A, C)，其中 H=E_K(0^128) 是认证密钥，J0 由 nonce 派生。nonce 复用的本质是：对于同一密钥，CTR 的密钥流分组和标签掩码 E_K(J0) 完全相同，导致密文异或等于明文异或，同时标签异或等于 GHASH 输出的异或。GHASH 碰撞在这里指：当 nonce 复用后，攻击者可以通过两条消息的密文/标签差异建立关于 H 的多项式方程，进而解出 H 或构造碰撞，完全突破完整性。该知识点处于现代密码学中 AEAD 的安全边界位置，是 TLS 1.3、IPsec、WireGuard 等协议安全论证的前提。专业工程师必须掌握，因为 nonce 管理错误是最常见且最危险的加密实现漏洞，且不会产生任何运行时告警。

### 2. 底层原理剖析
底层原理剖析：GHASH 处理流程是将 AAD 和密文各补零到 128 位倍数，末尾拼接 64 位 AAD 比特长度和 64 位密文比特长度。设输入分组 B1...Bm，H 为认证密钥，则 X0=0，Xi=(Xi-1 XOR Bi)*H，最后输出 Xm。展开后 Xm = XOR_{i=1..m} Bi * H^{m-i+1}，即每个分组对 H 的幂次贡献。所有运算均在 GF(2^128) 上，乘法是模不可约多项式 x^128+x^7+x^2+x+1 的二元无进位乘。
当 nonce 复用且两消息的 AAD/密文长度相同时，设标签为 T1,T2，由于 E_K(J0) 相同，T1 XOR T2 = GHASH(H,A1,C1) XOR GHASH(H,A2,C2)。GHASH 对输入分组是 XOR 线性的，对 H 是多项式；若长度相同，则 T1 XOR T2 = XOR_i (B1_i XOR B2_i) * H^{m-i+1}。令 Δ_i = B1_i XOR B2_i，得到一个关于 H 的多项式方程。如果 Δ_i 只有一个非零项，H 可直接开根求得；多个非零项时，可用有限域多项式求根算法（如 Cantor-Zassenhaus）解 H。一旦 H 已知，攻击者可以对任意 AAD/密文计算 GHASH，并用已知的 E_K(J0)（由一条已知消息的密文和标签反推）生成任意有效标签，实现完全伪造。
对比前端知识体系：这个机制与 React 列表渲染中 key 的唯一性要求有相似性。React 用 key 在多次渲染间建立旧/新节点的映射，key 重复会使 diff 算法错乱，导致组件状态错误挂载；AES-GCM 用 nonce 在多次加密间建立密钥流和认证掩码的映射，nonce 重复会使密文和标签的代数结构暴露。差别在于：React key 重复通常只是 UI 可观察的 bug，且开发环境会告警；GCM nonce 复用是密码学上的灾难，协议层无法感知，后果是机密性和完整性同时失效。因此 nonce 必须由单调计数器生成，或在无法保证唯一性的场景使用足够长的随机 nonce。

### 3. 基础代码与实战验证
```text
基础代码与实战验证：以下使用 cryptography 库演示 nonce 复用的直接后果，并给出恢复 H 的伪代码。
# 依赖: pip install cryptography
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

key = b'0' * 32          # 256-bit key
nonce = b'1' * 12        # 96-bit nonce，复用！
gcm = AESGCM(key)

p1 = b'attack at dawn!!'   # 16 字节
p2 = b'defend at dusk!!'   # 16 字节

c1 = gcm.encrypt(nonce, p1, None)  # 返回 ciphertext||tag
c2 = gcm.encrypt(nonce, p2, None)

ct1, tag1 = c1[:-16], c1[-16:]
ct2, tag2 = c2[:-16], c2[-16:]

# 验证: 同一 nonce 产生同一密钥流, 因此 ct1 xor ct2 == p1 xor p2
xor_ct = bytes(a ^ b for a, b in zip(ct1, ct2))
xor_pt = bytes(a ^ b for a, b in zip(p1, p2))
assert xor_ct == xor_pt  # 机密性已破

# 标签差异:
d = bytes(a ^ b for a, b in zip(tag1, tag2))
# d == GHASH(H, ct1) xor GHASH(H, ct2)，因为 E_K(J0) 抵消

# 攻击步骤 (伪代码):
# 设无 AAD、密文长度均为 16 字节 (1 个 128-bit 分组)，则 GHASH 展开为:
#   GHASH(H, C) = C * H^2 xor len_block * H
# 两条密文长度相同 -> len_block 相同，于是:
#   d = (C1 xor C2) * H^2
# 令 delta = C1 xor C2 (128-bit 分组)
# H^2 = d * delta^{-1}  (GF(2^128) 运算)
# H = (H^2)^(2^127)    # 有限域开平方: sqrt(x)=x^(2^(m-1))
# 得到 H 后，对任意伪造密文 C_fake，计算 tag_fake = E_K(J0) xor GHASH(H, C_fake)
# 其中 E_K(J0) = tag1 xor GHASH(H, ct1) 可由已知数据求出
```

### 4. 常见误区与进阶思考
常见误区与进阶思考：
误区 1：认为 nonce 复用只会泄露明文 XOR，不影响认证。实际上 nonce 复用同时使 E_K(J0) 掩码复用，标签方程变成关于 H 的已知多项式，攻击者可解出 H，从而伪造任意标签，完整性彻底失效。
误区 2：认为 GHASH 是像 SHA-256 一样的抗碰撞哈希。实际上 GHASH 是密钥化的通用哈希，只有在 H 保密时才有 2^-128 的碰撞概率；nonce 复用让 H 可解，碰撞可构造。不能将 GHASH 与加密哈希混为一谈。
思考题：攻击者只截获到两条同一 nonce 下的密文和标签，完全不知道任何明文。他是否已经足够恢复 H？如果能，请说明恢复 H 的过程中哪一步用到了明文；如果不能，缺少什么？
