---
title: "每日基础技术总结 · 2026-09-14 · RSA 的 OAEP 填充与 Bleichenbacher 攻击"
date: 2026-09-14 08:00:00
categories: [技术分享]
tags: ["技术分享", "安全基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-14 · RSA 的 OAEP 填充与 Bleichenbacher 攻击

## 📚 今日主题

> **RSA 的 OAEP 填充与 Bleichenbacher 攻击**（安全基础）

### 1. 核心概念速览
RSAES-OAEP 是 PKCS#1 v2.x 定义的 RSA 加密填充：把任意短消息 M 编码为与模数 n 等长的编码块 EM，再执行 RSA 原语 c = EM^e mod n。RSA 原语本身是确定性置换且乘法同态，直接加密会泄漏相等性并允许选择密文操纵；OAEP 通过哈希、MGF1、随机种子和 Feistel 式双掩码，将 M 随机化并加入可验证结构（lHash、PS、0x01），使解密方能识别编码是否合法。Bleichenbacher 攻击针对 PKCS#1 v1.5 加密：攻击者利用服务端是否返回“填充合法”的 padding oracle，结合 RSA 同态 c' = c·s^e mod n ⇒ m' = m·s mod n，自适应选择 s，把 m 的可能区间从 [2B,3B) 逐步收缩到唯一值，从而在约百万次查询内恢复明文。它属于选择密文攻击（CCA）与侧信道交叉领域，直接导致 SSL/TLS RSA key exchange、PKCS#1 v1.5 等大量协议漏洞（ROBOT）。在体系中，它位于公钥加密原语、填充编码、协议状态机与实现侧信道之间；前端工程师使用 Web Crypto 的 RSA-OAEP 时，底层正是该机制，若自造填充、区分解密错误或忽略时间差异，就会复现同类 oracle。

### 2. 底层原理剖析
底层机制分两层：
1) RSA 原语：m = c^d mod n，c = m^e mod n。它是确定性置换，满足 (m1·m2)^e = m1^e·m2^e mod n，因此可乘性：对密文 c 乘以 s^e，解出的是 m·s mod n。
2) PKCS#1 v1.5 加密编码：EM = 0x00 || 0x02 || PS || 0x00 || M，PS 至少 8 字节非零随机。解密后检查前两字节与 0x00 分隔符，合法返回 M，否则报错。这个“合法/非法”就是 oracle。
Bleichenbacher 攻击伪代码：
给定 c0，oracle O(c) = 1 当且仅当 RSA 解密 c 后符合 00 02 ...。
B = 2^(8(k-2))，初始区间 I = [2B, 3B)，因为合法 m 必落其中。
第一轮：找最小 s1 ≥ ceil(n/(3B)) 使 O(c0·s1^e mod n)=1。
后续：维护 m ∈ [a,b]。对每个候选 s，检查 m·s mod n 是否落入 [2B+t·n, 3B+t·n) 对某整数 t 成立；通过 oracle 筛选 s，反推新区间并交叠，直到区间唯一。
核心直觉：oracle 只泄漏 1 bit，但同态让攻击者能对同一 m 乘以任意 s，等价于在数轴上平移和缩放 m 的未知位置，反复二分/区间收缩。
OAEP 编码：
EM = 0x00 || maskedSeed || maskedDB
DB = lHash || PS || 0x01 || M，lHash = Hash(Label)
maskedDB = DB xor MGF1(seed, k-hLen-1)
maskedSeed = seed xor MGF1(maskedDB, hLen)
解码逆运算，校验 0x00、lHash、0x01 分隔符，失败必须返回同一错误。OAEP 使合法编码概率极低且与明文无关，但注意：Manger 攻击表明若 OAEP 解码泄漏中间步骤，仍可被选择密文攻击。
对比前端：PKCS#1 v1.5 类似后端接口先 JSON.parse 再校验字段，不同异常类型泄漏内部状态；Bleichenbacher 类似通过登录接口返回“用户不存在/密码错误”枚举账号，只是这里被枚举的是 RSA 明文的数值区间。OAEP 类似对请求体做带随机盐的规范化编码后再签名/加密，但验证必须在常量时间内完成。

### 3. 基础代码与实战验证
```text
import hashlib, os

def mgf1(seed, mask_len, hash_alg=hashlib.sha256):
    hLen = hash_alg().digest_size
    out = b''
    for counter in range((mask_len + hLen - 1) // hLen):
        out += hash_alg(seed + counter.to_bytes(4, 'big')).digest()
    return out[:mask_len]

def oaep_encode(message, k, label=b''):
    hLen = 32
    if len(message) > k - 2*hLen - 2: raise ValueError('message too long')
    lHash = hashlib.sha256(label).digest()
    PS = b'\x00' * (k - len(message) - 2*hLen - 2)
    DB = lHash + PS + b'\x01' + message
    seed = os.urandom(hLen)
    dbMask = mgf1(seed, k - hLen - 1)
    maskedDB = bytes(a ^ b for a, b in zip(DB, dbMask))
    seedMask = mgf1(maskedDB, hLen)
    maskedSeed = bytes(a ^ b for a, b in zip(seed, seedMask))
    return b'\x00' + maskedSeed + maskedDB

def oaep_decode(EM, k, label=b''):
    hLen = 32
    if len(EM) != k or EM[0] != 0: return None
    maskedSeed = EM[1:1+hLen]
    maskedDB = EM[1+hLen:]
    seedMask = mgf1(maskedDB, hLen)
    seed = bytes(a ^ b for a, b in zip(maskedSeed, seedMask))
    dbMask = mgf1(seed, k - hLen - 1)
    DB = bytes(a ^ b for a, b in zip(maskedDB, dbMask))
    if DB[:hLen] != hashlib.sha256(label).digest(): return None
    i = hLen
    while i < len(DB) and DB[i] == 0: i += 1
    if i == len(DB) or DB[i] != 1: return None
    return DB[i+1:]

# Bleichenbacher 攻击核心 oracle 伪代码：
# def oracle_v1_5(c, sk):
#     m = pow(c, sk.d, sk.n).to_bytes(sk.k, 'big')
#     return m[0:2] == b'\x00\x02'
# 攻击利用 c' = c * s^e mod n => m' = m * s mod n
# 每轮选 s，使合法概率最大化，用 oracle 筛选区间，最终恢复 m。
```

### 4. 常见误区与进阶思考
误区一：认为 OAEP 填充后 RSA 加密就无条件安全。OAEP 的证明依赖随机预言机模型，且只保护编码层；若实现泄漏任何可区分信息（不同错误串、不同耗时、不同长度、缓存访问模式），攻击者仍可构造选择密文攻击。典型如 Manger 攻击针对 OAEP 的 MSB oracle，ROBOT 针对 v1.5 变体。误区二：把 PKCS#1 v1.5 当作“旧但可用”的加密填充，或在 TLS/协议中自行处理解密失败。Bleichenbacher 的教训是：只要存在“填充合法/非法”的 1 bit oracle，结合 RSA 同态即可恢复明文；修复必须在协议层采用统一错误、常量时间、隐式拒绝、随机化 fallback，或直接使用 RSA-OAEP/KEM。误区三：混淆签名填充与加密填充，例如用 PKCS#1 v1.5 签名填充做加密，或用 OAEP 做签名。
思考题：若服务端对 OAEP 解密失败统一返回相同错误，但合法 OAEP 解码在找到 0x01 分隔符前会提前退出，而非法密文会扫描完整 DB，攻击者能否仅凭时间差构造 oracle？如果可以，如何利用 c' = c·s^e mod n 和已知的 OAEP 结构，把该时间差转化为对 m 的区间约束？
