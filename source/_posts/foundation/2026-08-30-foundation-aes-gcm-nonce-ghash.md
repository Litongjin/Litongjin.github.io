---
title: "每日基础技术总结 · 2026-08-30 · AES-GCM 的 nonce 复用与 GHASH 碰撞"
date: 2026-08-30 08:00:00
categories: [技术分享]
tags: ["技术分享", "安全基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-30 · AES-GCM 的 nonce 复用与 GHASH 碰撞

## 📚 今日主题

> **AES-GCM 的 nonce 复用与 GHASH 碰撞**（安全基础）

### 1. 核心概念速览
AES-GCM 是 AEAD（Authenticated Encryption with Associated Data）体制：AES-CTR 负责机密性，GHASH 负责完整性/真实性。GHASH 的本质是 GF(2^128) 上的多项式求值：先由 H = E_K(0^128) 得到认证子键 H，报文（AAD、密文、位长度块）逐块迭代 X_i = (X_{i-1} ⊕ block_i)·H（· 为 GF(2^128) 有限域乘法），标签 T = E_K(J0) ⊕ X_n。nonce 同时决定 CTR 密钥流（J0 自增）与标签掩码 E_K(J0)，是单点故障。nonce 复用的本质：同一密钥 + 同一 nonce ⇒ 相同 H、相同掩码、相同密钥流。后果有两层：(1) 机密性层面，C1⊕C2 = P1⊕P2；(2) 完整性层面，两条标签异或后消去 E_K(J0)，得到关于 H 的确定多项式方程，已知明文或 AAD 即可求解 H，从而对任意密文伪造合法标签。这不是『两个标签恰好相等』的概率碰撞，而是确定性代数冗余——nonce 复用把认证密钥直接暴露在有限域方程中。位置：密码学 → 对称加密 → AEAD，是 TLS 1.3、QUIC、JWE、AWS Encryption SDK 的事实基础原语。专业工程师必须掌握，因为 nonce 分配是工程决策（全局计数器、并发分配、密钥轮换），违约时不抛异常、不报错，只在 GF(2^128) 中静默坍缩为全线失陷。

### 2. 底层原理剖析
一、GCM 内部结构（96 位 nonce 情形）：
  H = AES_K(0^128)
  J0 = nonce || 0x00000001
  加密密钥流首块 = AES_K(inc32(J0)) = AES_K(nonce || 0x00000002)，后续计数器依次递增
  标签掩码 = AES_K(J0)
  GHASH 输入 = pad16(A) || pad16(C) || [bitlen(A) 的 64 位大端 || bitlen(C) 的 64 位大端]
  递推：X_0 = 0；X_i = (X_{i-1} ⊕ block_i) · H；输出 X_n
  最终标签 T = AES_K(J0) ⊕ X_n

二、nonce 复用的代数展开（核心）：
设两次加密使用同一 key 与同一 nonce，明文 P1、P2 已知。
密钥流相同：KS = C1⊕P1 = C2⊕P2，所以 C1⊕C2 = P1⊕P2——机密性直接坍塌。
标签方面：T1 = M ⊕ GHASH_H(C1)，T2 = M ⊕ GHASH_H(C2)，其中 M = AES_K(J0) 相同，于是 T1⊕T2 = GHASH_H(C1) ⊕ GHASH_H(C2)。
若两条消息均为单块（16 字节）且无 AAD，则 GHASH 展开为 GHASH_H(C) = C·H^2 ⊕ L·H（L 为位长度块）。两次长度块 L 相同、二次项抵消，得：
  D = T1⊕T2 = (C1⊕C2)·H^2
这是 GF(2^128) 中关于 H 的一元方程。C1⊕C2 已知且几乎必然可逆，于是 H^2 = D·(C1⊕C2)^{-1}。
在 GF(2^128) 中平方根运算就是 2^127 次幂（Frobenius 自同构的逆）：H = R^(2^127)，因为 H^(2^128) = H。
得到 H 后，用任一已知 (C1, T1) 反解掩码 M = T1 ⊕ GHASH_H(C1)，此后对任意密文 C' 计算 T' = M ⊕ GHASH_H(C') 即为合法标签——完全伪造；同时该 nonce 下所有历史消息的密钥流已知，可全部解密。

三、与前端已有概念的异同：nonce 的唯一性约束不是『风格规范』而是『代数前提』。如同 Java 接口是名义类型契约、TS 接口是结构类型契约——两者都叫接口，但违约发生的位置与表现完全不同；nonce 在报文格式层只是一个明文字段，但在 GCM 代数层它同时决定密钥流与标签掩码，违约不在字段层报错，而在 GF(2^128) 中坍缩为可解方程。再如前端 Map 的哈希碰撞由链地址法兜底，GCM 没有『碰撞解决』机制——nonce 别名直接将多次加密的方程叠加，把认证密钥 H 从未知数变成可求根的目标；React 列表 key 复用最多造成状态错位（启发式问题），GCM nonce 复用造成确定性密钥恢复（数学问题），量级完全不同。

### 3. 基础代码与实战验证
```text
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import os

# ---------- GF(2^128)：不可约多项式 x^128+x^7+x^2+x+1 ----------
MASK = (1 << 128) - 1
RED = 0x87  # x^7+x^2+x+1 的低位系数

def gf_mul(x, y):
    """GF(2^128) 乘法：逐位扫描 y；x 每次左移一位表示乘多项式 t，
    最高位溢出时异或 RED 完成模约简。"""
    z = 0
    for _ in range(128):
        if y & 1:
            z ^= x
        carry = (x >> 127) & 1   # 取当前 x 的最高位，决定是否降次
        x = (x << 1) & MASK
        if carry:
            x ^= RED
        y >>= 1
    return z

def gf_pow(a, e):
    """平方-乘求幂；求逆用 a^(2^128-2)，开方用 a^(2^127)"""
    r = 1
    while e:
        if e & 1:
            r = gf_mul(r, a)
        a = gf_mul(a, a)
        e >>= 1
    return r

def aes_block(key, blk):
    """AES 单块加密（ECB 模式）"""
    return Cipher(algorithms.AES(key), modes.ECB()).encryptor().update(blk)

def gh(H, aad, ct):
    """GHASH：AAD 与密文各补零到 16 字节块，末块为两者的 64 位位长度；
    每块迭代 X=(X⊕block)·H。"""
    def pad16(b):
        return b + b'\x00' * ((16 - len(b) % 16) % 16)
    data = pad16(aad) + pad16(ct) + \
           (8 * len(aad)).to_bytes(8, 'big') + (8 * len(ct)).to_bytes(8, 'big')
    X = 0
    for i in range(0, len(data), 16):
        X = gf_mul(X ^ int.from_bytes(data[i:i+16], 'big'), H)
    return X

def gcm_enc(key, nonce, pt, aad):
    """自制 GCM：H=E_K(0)，掩码 M=E_K(J0)，密钥流从 J0+1 起；
    标签 = M ⊕ GHASH_H(ciphertext)"""
    H = int.from_bytes(aes_block(key, b'\x00'*16), 'big')   # 认证子键
    J0 = nonce + b'\x00\x00\x00\x01'
    M = int.from_bytes(aes_block(key, J0), 'big')            # 标签掩码
    ct = b''
    for i in range(0, len(pt), 16):
        ks = aes_block(key, nonce + (2 + i // 16).to_bytes(4, 'big'))  # CTR 计数从 2 起
        ct += bytes(p ^ k for p, k in zip(pt[i:i+16], ks))
    return ct, (M ^ gh(H, aad, ct)).to_bytes(16, 'big')

# 先验证自制实现与标准库等价，确保后续攻击是标准库可复现的真实漏洞
key = b'K' * 16
n2, p_rand = os.urandom(12), os.urandom(64)
ref = AESGCM(key).encrypt(n2, p_rand, b'aad')
assert gcm_enc(key, n2, p_rand, b'aad') == (ref[:-16], ref[-16:])

# ---------- 攻击：同 key 同 nonce 加密两条 16 字节明文 ----------
nonce = b'V' * 12
p1, p2 = b'A' * 16, b'B' * 16
c1, t1 = gcm_enc(key, nonce, p1, b'')
c2, t2 = gcm_enc(key, nonce, p2, b'')

ks = bytes(a ^ b for a, b in zip(c1, p1))            # 从已知明文反推密钥流
assert bytes(a ^ b for a, b in zip(c2, p2)) == ks    # 证明密钥流被完全复用

X = int.from_bytes(bytes(a ^ b for a, b in zip(c1, c2)), 'big')
D = int.from_bytes(bytes(a ^ b for a, b in zip(t1, t2)), 'big')
R = gf_mul(D, gf_pow(X, (1 << 128) - 2))             # D/X = H^2（长度块抵消）
Hv = R
for _ in range(127):                                 # 开方：H=(H^2)^(2^127)
    Hv = gf_mul(Hv, Hv)
assert Hv == int.from_bytes(aes_block(key, b'\x00'*16), 'big')  # 认证子键 H 已被恢复

M = int.from_bytes(t1, 'big') ^ gh(Hv, b'', c1)      # 反解掩码 E_K(J0)
p3 = b'FORGED-GHASH-!!!'                             # 任意伪造明文
c3 = bytes(p ^ k for p, k in zip(p3, ks))            # 用复用密钥流生成对应密文
T3 = (M ^ gh(Hv, b'', c3)).to_bytes(16, 'big')       # 不依赖 AES 密钥造出合法标签
assert AESGCM(key).decrypt(nonce, c3 + T3, b'') == p3  # 受害者的解密接口接受伪造
print('H 已恢复，伪造密文被标准库接受：', p3)
```

### 4. 常见误区与进阶思考
误区一：『nonce 用随机生成即可，碰撞概率可忽略』。96 位随机 nonce 在 2^32 次加密时碰撞概率即约 2^-33（生日界 n^2/2^97），2^40 次时约 2^-17；高 QPS 的网关/微服务几年内即可触达边界。随机 nonce 只适合短生存期密钥；工程上应使用进程/实例内原子递增计数器，或由上层协议显式分配唯一 nonce，并在密钥轮换时重置计数。

误区二：『nonce 复用只是重复密钥流，最多泄露明文异或；AES 没破就还能保住完整性』。事实是标签方程直接泄露认证子键 H：两条同 nonce 且已知明文/已知 AAD 的消息足以解出 H，攻击者随后可对任意密文伪造标签。这里的『GHASH 碰撞』不是两个标签偶然相等的概率事件，而是 nonce 复用导致的代数方程坍缩——认证与加密共用 nonce，完整性保护在 nonce 违约时与机密性一起失效。

思考题：若两次复用 nonce 的消息块数不同（C1 为 1 块、C2 为 2 块），长度块不再抵消。写出 T1⊕T2 关于 H 的多项式方程（形如 (C2_1)·H^3 ⊕ (C1⊕C2_2)·H^2 ⊕ (L1⊕L2)·H = T1⊕T2），并说明已知明文时如何把 H 的求解化为 GF(2^128) 上的多项式求根问题；为什么这个攻击仍然不触及 AES 密钥本身？
