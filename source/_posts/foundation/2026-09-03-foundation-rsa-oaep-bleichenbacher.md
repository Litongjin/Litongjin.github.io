---
title: "每日基础技术总结 · 2026-09-03 · RSA 的 OAEP 填充与 Bleichenbacher 攻击"
date: 2026-09-03 08:00:00
categories: [技术分享]
tags: ["技术分享", "安全基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-03 · RSA 的 OAEP 填充与 Bleichenbacher 攻击

## 📚 今日主题

> **RSA 的 OAEP 填充与 Bleichenbacher 攻击**（安全基础）

### 1. 核心概念速览
RSA-OAEP（Optimal Asymmetric Encryption Padding，最优非对称加密填充）是 RSA 加密中用于将确定性算法随机化的填充方案，由 Bellare 与 Rogaway 于 1994 年提出，基于 Fujisaki-Okamoto 变换思想，将任意长度的消息通过 Feistel 网络与随机种子混合，产生密文空间的均匀分布，从而在随机预言机模型下提供 IND-CCA2 安全性。它解决的根本问题是：原始 RSA（教科书式 RSA）是确定性且不具备密文不可延展性，攻击者可通过数学性质（如乘法同态）在不解密的情况下篡改密文或恢复明文。OAEP 通过引入随机性和可验证的冗余结构，阻断 Bleichenbacher 攻击所需的 oracle 条件。Bleichenbacher 攻击（1998 年）是针对 PKCS#1 v1.5 填充的适应性选择密文攻击：攻击者利用服务器返回的“填充是否合法”的侧信道（padding oracle），通过反复修改密文并观察响应，以多项式时间将密文解密为明文，复杂度约 2^20 次查询。该攻击是 RSA 安全模型演进的里程碑，直接促使 OAEP 成为标准填充。在计算机体系中，它是公钥密码学、可证明安全理论、侧信道攻击交叉领域的核心案例；专业工程师必须掌握，因为 TLS 1.2 及早期协议中 RSA 密钥交换仍可能受此影响，且任何使用 RSA 加密的场景（如密钥封装、数字信封）若不使用 IND-CCA2 安全填充，都会重现此类漏洞。理解它需要同时具备数论基础（RSA 同态性质）、概率论（随机预言机）与工程视角（oracle 侧信道）。

### 2. 底层原理剖析
OAEP 的底层机制本质是一个两轮 Feistel 网络。设 k 为 RSA 模数长度（字节），消息 m 长度为 k - 2hLen - 2，随机种子 r 长度为 hLen（哈希输出长度）。填充过程：1) 构造数据块 DB = lHash || PS || 0x01 || m，其中 lHash 为标签的哈希，PS 为若干 0x00 字节；2) 计算 maskedDB = DB ⊕ MGF(r)，MGF 是掩码生成函数（通常基于哈希的计数器模式）；3) 计算 maskedSeed = r ⊕ MGF(maskedDB)；4) 输出 EM = 0x00 || maskedSeed || maskedDB，作为 RSA 的输入 x。解密时逆向操作，先剥离 0x00，恢复 maskedSeed 和 maskedDB，再计算 r = maskedSeed ⊕ MGF(maskedDB)，DB = maskedDB ⊕ MGF(r)，最后检查 lHash 和 0x01 分隔符是否匹配，若不匹配则返回“填充错误”。这种结构使得对密文的任何篡改都会在解密后以压倒性概率导致冗余校验失败。Bleichenbacher 攻击的原理利用的是 RSA 的乘法同态：给定目标密文 c = m^e mod n，攻击者选择因子 s，计算 c' = c * s^e mod n = (m*s)^e mod n。若服务器作为 oracle 告知 c' 解密后的填充是否合法（PKCS#1 v1.5 中要求明文以 0x00 0x02 开头），攻击者就能逐步缩小 m 的区间。通过巧妙构造 s 并观察 oracle 响应，可将 m 的候选区间从 [0, n) 二分压缩到唯一值。攻击的核心是 oracle 泄露了填充结构的一比特信息（是否合法），结合 RSA 同态性形成自适应区间收缩。与前端对比：前端工程师熟知的 Java 接口与 TS 接口的区别在于 Java 接口是编译期类型契约且可被实现类强制遵守，TS 接口是结构类型系统的运行时擦除；这里的本质差异是“可验证的冗余”与“不可验证的随机性”。PKCS#1 v1.5 填充相当于弱类型检查——只有高位字节的粗粒度校验，攻击者能利用同态构造大量合法样本；OAEP 相当于强类型检查——通过 Feistel 网络将消息与随机种子深度混合，任何比特翻转都会在解密后触发哈希校验失败，且校验失败的信息不泄露任何关于明文的统计特征。换句话说，Bleichenbacher 攻击之所以成立，是因为 v1.5 填充的“合法性谓词”是单调的、区间性的（0x00 0x02 开头），而 OAEP 的“合法性谓词”是伪随机的、不可预测的。

### 3. 基础代码与实战验证
```text
以下为 Python 伪代码，展示 OAEP 填充的核心机制与 Bleichenbacher 攻击的 oracle 模拟，不依赖密码学库，仅用于理解原理。

# ============ OAEP 填充（简化，使用 SHA-256 作为哈希与 MGF）============
import hashlib, os, math

def mgf(seed, length):
    """掩码生成函数：用哈希计数器扩展种子到指定长度"""
    out = b''
    counter = 0
    while len(out) < length:
        out += hashlib.sha256(seed + counter.to_bytes(4, 'big')).digest()
        counter += 1
    return out[:length]

def oaep_pad(m, k=256, hLen=32):
    """OAEP 编码：m 为消息，k 为模数长度（字节），hLen 为哈希长度"""
    lHash = hashlib.sha256(b'').digest()  # 标签为空
    if len(m) > k - 2*hLen - 2:
        raise ValueError("消息过长")
    PS = b'\x00' * (k - len(m) - 2*hLen - 2)  # 填充零
    DB = lHash + PS + b'\x01' + m  # 数据块 = 标签哈希 || 零 || 分隔符 || 消息
    r = os.urandom(hLen)  # 随机种子
    maskedDB = bytes(a ^ b for a, b in zip(DB, mgf(r, k - hLen - 1)))
    maskedSeed = bytes(a ^ b for a, b in zip(r, mgf(maskedDB, hLen)))
    return b'\x00' + maskedSeed + maskedDB  # 最终编码结果

def oaep_unpad(EM):
    """OAEP 解码：验证冗余，返回消息或抛出异常（即 oracle 的判定）"""
    if EM[0] != 0:
        return None
    maskedSeed = EM[1:1+32]
    maskedDB = EM[33:]
    r = bytes(a ^ b for a, b in zip(maskedSeed, mgf(maskedDB, 32)))
    DB = bytes(a ^ b for a, b in zip(maskedDB, mgf(r, len(maskedDB))))
    lHash = DB[:32]
    if lHash != hashlib.sha256(b'').digest():
        return None
    # 查找 0x01 分隔符（实际标准要求严格格式，此处简化）
    idx = DB.find(b'\x01', 32)
    if idx == -1:
        return None
    return DB[idx+1:]

# ============ Bleichenbacher 攻击的 oracle 模拟 ============
# 教科书式 RSA 加解密：c = m^e mod n
# 假设服务器使用 PKCS#1 v1.5 填充（非 OAEP）解密，并泄露“填充是否合法”

def pkcs15_oracle(c):
    """模拟服务器：解密 c 后检查明文是否以 0x00 0x02 开头，返回布尔值"""
    m = pow(c, d, n)  # d 为私钥指数
    plaintext = m.to_bytes(k, 'big')
    return plaintext[0] == 0 and plaintext[1] == 2

# 攻击者拥有公钥 (e, n) 和目标密文 c，目标是恢复 m = c^d mod n
# 核心步骤：
# 1. 寻找 s 使得 c' = c * s^e mod n 的解密结果以 0x00 0x02 开头（合法）
# 2. 根据合法 s 的信息，将 m 的候选区间缩小
# 3. 重复直到区间内只有一个整数，即明文 m
# 攻击伪代码：
# s = 1
# while True:
#     c_prime = (c * pow(s, e, n)) % n
#     if pkcs15_oracle(c_prime):
#         update_interval(m, s)  # 根据 s 和 oracle 响应收缩区间
#         if interval_size == 1:
#             return m
#     s += 1
# 注意：实际攻击需要精心构造 s 的更新策略（如 Bleichenbacher 的三种情况），
# 但本质就是利用 oracle 泄露的填充合法性，结合 RSA 同态性进行区间收缩。

# 对比：若使用 OAEP，oracle 返回的是“OAEP 校验是否通过”，
# 该布尔值在随机预言机模型下与消息的代数结构无关，
# 攻击者无法从合法/非法响应中推导出关于 m 的区间信息，因此攻击失效。

# 关键注释：OAEP 的 Feistel 结构使得对 c 的任何代数变换（乘 s^e）
# 在解密后导致消息与随机种子都被彻底打乱，冗余校验失败的概率约为 1 - 2^{-hLen}，
# 且失败模式不呈现任何与 m 相关的统计规律，从而阻断区间收缩。
```

### 4. 常见误区与进阶思考
误区一：认为只要使用 RSA 且填充格式正确就安全。实际上 PKCS#1 v1.5 填充在 1998 年就被证明不安全，直到 TLS 1.3 才彻底移除 RSA 密钥交换。即便在 TLS 1.2 中，服务器必须对解密失败返回统一错误，且要求使用恒定时间比较，否则仍可能被 Bleichenbacher 变体攻击（如 ROBOT 攻击）。正确认知是：填充的安全性不仅取决于格式，更取决于解密失败时的行为是否泄露可区分信息。OAEP 的安全性建立在随机预言机模型上，实际实现中必须确保 MGF 与哈希的选择、错误处理的一致性、以及侧信道（时间、功耗）的防护，否则可证明安全也徒劳。

误区二：混淆 RSA 加密与 RSA 签名的填充。OAEP 是加密填充，PSS 是签名填充，两者不可互换。签名场景不需要 IND-CCA2 但需要不可伪造性，而加密场景必须抵抗选择密文攻击。工程师常误以为用 OAEP 加密、用 PKCS#1 v1.5 签名没问题，但签名时使用的填充若存在 oracle（如验证错误信息泄露），同样可能导致密钥恢复（如 Bleichenbacher 攻击对 PKCS#1 v1.5 签名的变体）。

思考题：在 Bleichenbacher 攻击中，假设服务器在解密失败时返回的 HTTP 状态码不同（200 vs 500），攻击者能否利用该信息？进一步，如果将 RSA 的指数 e 从 65537 改为 3，攻击的查询次数或复杂度会如何变化？请从 oracle 的信息论容量和 RSA 同态性质推导，而不是凭直觉回答。
