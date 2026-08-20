---
title: "每日基础技术总结 · 2026-05-26 · AES-GCM 的 nonce 复用与 GHASH 碰撞"
date: 2026-05-26 08:00:00
categories: [技术分享]
tags: ["技术分享", "安全基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-05-26 · AES-GCM 的 nonce 复用与 GHASH 碰撞

## 📚 今日主题

> **AES-GCM 的 nonce 复用与 GHASH 碰撞**（安全基础）

### 1. 核心概念速览
AES-GCM 是一种 AEAD 加密方案，由 CTR 模式加密与 GHASH（基于 GF(2^128) 的通用散列）两部分组成。GHASH 的密钥 H = AES_K(0^128) 是固定不变的。Nonce 在 GCM 中仅用于生成初始计数器，并直接参与 GHASH 对附加数据、密文和长度块的认证。Nonce 复用意味着在同一密钥下，两个加密过程使用了相同的初始计数器，这直接导致认证标签的密钥流（即 GHASH 内部的多项式求值）出现代数关联。本质是：GHASH 是一个基于 H 的 2-adic 多项式，当两个消息使用同一 nonce 时，它们的标签差 T1⊕T2 等于一个以 H 为未知量的多项式方程，攻击者可通过求解该方程（在 GF(2^128) 上因式分解）恢复 H。一旦 H 泄露，攻击者即可伪造任意消息的合法标签，彻底破坏 GCM 的认证性，进而可实施密文篡改、密钥流恢复甚至明文还原。该知识点位于对称密码学、认证加密与侧信道/误用分析的交汇处，是专业工程师设计安全协议（如 TLS 1.3、QUIC、JWE）时必须刻入骨髓的约束：nonce 绝对不可重复。

### 2. 底层原理剖析
GCM 加密：将 nonce || counter(32-bit) 作为 CTR 的输入块，E_K 加密后与明文异或得密文。认证部分：将附加数据 A、密文 C 按 128-bit 分块，末尾拼接长度块（len(A)||len(C)），构成序列 X_1,...,X_m。GHASH 定义为 Y_i = (Y_{i-1} ⊕ X_i) · H，其中 H = E_K(0^128)，乘法为 GF(2^128) 上的多项式乘法模不可约多项式 x^128 + x^7 + x^2 + x + 1。最终标签 T = E_K(J_0) ⊕ Y_m，其中 J_0 = nonce || 0^31 || 1。设两次加密使用同一 nonce，密钥相同，则 H 相同，J_0 相同，因此 E_K(J_0) 相同。设两消息的 GHASH 输出分别为 Y 和 Y'，则 T⊕T' = Y⊕Y'。展开 Y = ⊕_{i=1}^m (X_i · H^{m-i+1})（注意顺序），Y' 同理。两式相加得一个关于 H 的多项式方程：⊕_{i} (X_i⊕X'_i)·H^{k_i} = T⊕T'。由于消息内容（至少密文块）已知，这是 GF(2^128) 上的一个多项式方程。其根集合有限，攻击者只需对 H 的可能值做因式分解（实际可用更高效的方法，如计算多项式 gcd），即可在多项式时间内恢复 H。对比前端知识：这类似于两个不同输入经过同一个哈希函数后产生差分，但 GHASH 是线性结构（在 XOR 意义下），而普通哈希（如 SHA-256）是非线性混淆，无法从差分反推内部状态。GCM 的认证本质是一个带密钥的线性映射，线性意味着代数攻击可行。前端工程师熟悉的事件循环回调顺序、React 渲染不可变状态等概念，都强调不可变性和确定性，但 GCM 的 nonce 复用问题更接近『两个对象共享同一个内部 ID』导致的身份混淆——不过这里是密码学层面的灾难性泄漏。

### 3. 基础代码与实战验证
```text
// 以下为概念验证代码（Python + cryptography 库），演示 nonce 复用如何泄露 GHASH 密钥 H。
// 实际攻击不直接输出 H，但通过两个已知明文/密文/标签即可构造方程，此处用简化方式展示核心代数关系。

from cryptography.hazmat.primitives.ciphers.aead import AESGCM
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
import os

key = os.urandom(32)
nonce = b"\x00"*12  # 故意复用同一个 nonce

# 第一次加密（已知明文 m1）
aesgcm = AESGCM(key)
m1 = b"attack at dawn"
ct1 = aesgcm.encrypt(nonce, m1, None)

# 第二次加密（已知明文 m2）
m2 = b"retreat at noon"
ct2 = aesgcm.encrypt(nonce, m2, None)

# 假设攻击者截获 ct1, ct2，且已知 m1, m2（例如常见协议头）。
# 由于 CTR 模式密钥流相同，可得到密钥流 K = ct1 ^ m1 == ct2 ^ m2。
keystream = bytes(a ^ b for a, b in zip(ct1, m1))

# 但更关键的是认证标签。计算 GHASH 的输入块（此处略去长度块，实际攻击需完整构造）
# 设 Y = GHASH(H, A, C)。标签 T = E_K(J0) ^ Y。
# 两次标签 T1, T2 的差 = Y1 ^ Y2。
# Y1 ^ Y2 是关于 H 的多项式：
# 对每个 128-bit 块，有 (C1_i ^ C2_i) * H^{n-i+1} 的 XOR。
# 攻击者已知 C1_i 和 C2_i，以及 T1^T2，因此得到一个关于 H 的方程。
# 实际求解 H 使用多项式因式分解，这里不展开。

# 下面展示如何验证：若 H 已知，可以伪造任意消息的标签。
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.backends import default_backend

# 计算 H = E_K(0^128)
backend = default_backend()
cipher = Cipher(algorithms.AES(key), modes.ECB(), backend=backend)
encryptor = cipher.encryptor()
H = encryptor.update(b"\x00"*16)  # AES_K(0^128)

# 伪造消息：构造 m_fake，使用同样的 nonce 加密（CTR 密钥流已知）
# 实际上攻击者可构造任意密文块和附加数据，并计算对应 GHASH 和标签。
# 这里仅演示 H 的计算，真实攻击者不会需要这个步骤，而是直接解方程。

# 结论：nonce 复用使认证系统崩溃，因为 H 可通过代数方程求解。
```

### 4. 常见误区与进阶思考
常见误区一：认为 nonce 复用只泄露密钥流，导致机密性受损，但认证性仍然安全。实际恰恰相反，nonce 复用的最严重后果是 GHASH 密钥 H 被恢复，攻击者可以完全伪造任意消息的标签，即认证性彻底崩溃。即使消息内容未知（例如只有密文），攻击者只要获得两次相同 nonce 的密文和标签，且能猜测或知道部分明文（如协议固定头），就能构造方程恢复 H。

常见误区二：认为使用随机 nonce 就可以安全，忽略随机 nonce 碰撞概率。对于 96-bit nonce，当加密数量达到约 2^48 时，碰撞概率接近 50%（生日界）。所以实际协议中必须使用计数器或状态维护保证 nonce 唯一，或采用更大 nonce（如 192-bit）并分层设计。TLS 1.3 和 QUIC 使用隐式 64-bit 序号作为 nonce 的一部分，就是为了从协议层面杜绝复用。

思考题：给定两次使用同一 nonce 的 GCM 加密，已知附加数据 A1、A2，密文 C1、C2，标签 T1、T2，但明文未知。请证明你能恢复 GHASH 密钥 H（提示：GHASH 的输入块中，明文只影响密文块，但密文块是已知的；而 CTR 密钥流未知，但两个密文块的差等于两个明文的差，这个差在 GHASH 方程中如何消去？注意 GHASH 的输入是密文块而不是明文块，因此实际上你不需要知道明文——密文块本身就是已知的。那么攻击者还需要什么条件？请推导出恢复 H 所需的方程个数和求解步骤。）
