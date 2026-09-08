---
title: "每日基础技术总结 · 2026-09-08 · RSA 的 OAEP 填充与 Bleichenbacher 攻击"
date: 2026-09-08 08:00:00
categories: [技术分享]
tags: ["技术分享", "安全基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-08 · RSA 的 OAEP 填充与 Bleichenbacher 攻击

## 📚 今日主题

> **RSA 的 OAEP 填充与 Bleichenbacher 攻击**（安全基础）

### 1. 核心概念速览
OAEP（Optimal Asymmetric Encryption Padding）是 RSA 加密中用于将确定性的 RSA 原始操作转化为概率性、语义安全的加密方案的填充机制。其本质是在明文上叠加 Feistel 网络结构的掩码，使得相同明文在不同随机种子下产生不同密文，并强制引入可验证的冗余码，从而阻断利用 RSA 乘法同态性进行的选择密文攻击。Bleichenbacher 攻击是 1998 年针对 RSA 加密中 PKCS#1 v1.5 填充的 padding oracle 攻击，它利用服务器对填充有效性（尤其 0x00 0x02 前缀）的不同响应作为旁路通道，通过密文乘方变换逐步逼近明文。该知识点位于公钥密码学、可证明安全与侧信道安全的交叉点，是理解 RSA 在实际协议（如 TLS）中为何必须使用 OAEP 或 RSAES-PKCS1-v1_5 并严格统一错误响应的核心。专业工程师必须掌握，因为现代 HTTPS/安全协议中的错误设计、时间侧信道、oracle 攻击等现实威胁均源于此类填充层的细微漏洞。

### 2. 底层原理剖析
RSA 原始操作是确定性双射：c = m^e mod n，m = c^d mod n。无填充时，同一明文永远对应同一密文，且具有乘法同态（E(m1)E(m2) = E(m1m2)），攻击者可利用这些性质实施选择密文攻击。OAEP 填充通过两轮 Feistel 变换引入随机性。其构造如下：

1. 设 k 为 RSA 模长字节数，mLen 为明文长度，hLen 为哈希输出长度。
2. 构造数据块 DB = lHash || PS || 0x01 || M，其中 lHash 是标签（如空串）的哈希，PS 为全 0 填充，长度保证 DB 总长为 k - hLen - 1。
3. 生成随机种子 seed（长度 hLen）。
4. 计算 maskedDB = DB ⊕ MGF(seed)，其中 MGF 是基于哈希的掩码生成函数（通常为 MGF1，即哈希计数器循环）。
5. 计算 maskedSeed = seed ⊕ MGF(maskedDB)。
6. 最终填充块 EM = 0x00 || maskedSeed || maskedDB，作为 RSA 加密输入。

解密时逆序执行：先计算 maskedSeed 与 maskedDB，再恢复 seed 与 DB，验证 lHash 与 0x01 分隔符。任何失败均视为填充无效。

Bleichenbacher 攻击针对 PKCS#1 v1.5 填充（格式 0x00 0x02 || PS || 0x00 || M，PS 至少 8 字节非零）。攻击者截获密文 c，构造 c' = c * s^e mod n，发送给服务器；若服务器响应“填充有效”（对应 s 使得 m*s mod n 的前两字节为 0x00 0x02），则攻击者获得关于 m 的区间信息。通过不断调整 s，将可能的明文区间二分缩小，最终恢复 m。其本质是利用 RSA 乘法性质将密文变换为与合法填充相关的值，并以服务器填充检查作为 oracle。攻击复杂度约为 2^20 次查询（对 1024 位 RSA），远低于直接分解。

与前端概念对比：类似前端中“输入校验”与“后端二次校验”的关系——前端校验只提升用户体验，不构成安全边界；RSA 中若将填充检查结果（成功/失败）暴露给网络对端，就如同后端将“数据库唯一键冲突”的详细信息返回给前端，为攻击者提供了可利用的预言机。又好比 TypeScript 的接口约束只在编译期存在，运行时消失；OAEP 的冗余校验是运行时强制执行的语义完整性检查，必须与错误响应统一，否则就产生隐蔽信道。

### 3. 基础代码与实战验证
以下用最小化 Python 代码验证 OAEP 的核心 Feistel 结构与 Bleichenbacher 攻击的基本原理。不依赖框架，仅使用 hashlib 模拟 MGF。

```python
import hashlib, random, os

def mgf1(seed, length):
    """MGF1：用 SHA-256 生成任意长度掩码"""
    out = b""
    counter = 0
    while len(out) < length:
        out += hashlib.sha256(seed + counter.to_bytes(4, 'big')).digest()
        counter += 1
    return out[:length]

def oaep_pad(message):
    """OAEP 填充（简化：忽略标签哈希，用固定 lHash）"""
    k = 128      # 模长 1024 位 = 128 字节
    hLen = 32    # SHA-256 输出长度
    mLen = len(message)
    assert mLen <= k - 2 * hLen - 2
    lHash = b"\x00" * hLen  # 实际应为标签哈希，此处简化为 0
    PS = b"\x00" * (k - mLen - 2 * hLen - 2)
    DB = lHash + PS + b"\x01" + message
    seed = os.urandom(hLen)
    maskedDB = bytes(a ^ b for a, b in zip(DB, mgf1(seed, k - hLen - 1)))
    maskedSeed = bytes(a ^ b for a, b in zip(seed, mgf1(maskedDB, hLen)))
    return b"\x00" + maskedSeed + maskedDB

def oaep_unpad(em):
    """OAEP 解填充：验证冗余；返回明文或抛错"""
    hLen = 32
    maskedSeed = em[1:1+hLen]
    maskedDB = em[1+hLen:]
    seed = bytes(a ^ b for a, b in zip(maskedSeed, mgf1(maskedDB, hLen)))
    DB = bytes(a ^ b for a, b in zip(maskedDB, mgf1(seed, len(maskedDB))))
    lHash = DB[:hLen]
    if lHash != b"\x00" * hLen:
        raise ValueError("invalid padding")
    i = hLen
    while i < len(DB) and DB[i] == 0:
        i += 1
    if i >= len(DB) or DB[i] != 1:
        raise ValueError("invalid padding")
    return DB[i+1:]

# 验证：填充可逆且随机
text = b"attack at dawn"
p1 = oaep_pad(text)
p2 = oaep_pad(text)
print("随机不同:", p1 != p2)
print("恢复一致:", oaep_unpad(p1) == text)

# Bleichenbacher 攻击核心步骤（概念验证，不实现完整二分）
# 假设 c = m^e mod n，攻击者发送 c' = c * s^e mod n
# 服务器返回 oracle(b) = True 当解密后填充有效（即前两字节 0x00 0x02）
# 模拟 oracle：
def pkcs1_v15_pad_ok(m):
    # 实际中检查 m 的字节表示是否以 0x00 0x02 开头且含 0x00 分隔符
    return len(m) >= 3 and m[0] == 0 and m[1] == 2

# 攻击者的迭代：调整 s 使 m*s mod n 落在某个区间内
# 具体数学推导涉及区间交集，此处仅展示 oracle 查询模式
s = random.randrange(2**16, 2**17)
# 攻击者计算 c_prime = (c * pow(s, e, n)) % n，并发送给服务器
# 服务器解密得到 m_prime = (m * s) % n，填充是否有效作为回答
# 通过多次 s 的更新，缩小 m 的候选区间，最终唯一确定明文。
```

注意：实际 Bleichenbacher 攻击需要数百次至数万次 oracle 查询，并需处理边界条件与中值步骤。以上代码仅揭示其依赖的数学变换与 oracle 响应模式。

### 4. 常见误区与进阶思考
误区一：认为使用 RSA-OAEP 后就可以随意向客户端返回“解密失败”或“填充错误”的细节。实际上，即使 OAEP 本身是 IND-CCA2 安全的，但如果应用层将区分“填充无效”与“其他错误”（或在时间上产生可测量差异），则仍可能形成 padding oracle。Bleichenbacher 攻击的本质是 oracle 存在，而非填充算法本身。正确做法是统一错误响应并加入恒定时间操作。

误区二：混淆“概率加密”与“随机性填充”。OAEP 的随机性并非单纯为了产生不同密文，而是为了将 RSA 从 trapdoor permutation 提升为具有可证明安全性的加密 scheme。若随机种子可预测或 MGF 实现错误，则语义安全被破坏。

思考题：若某 RSA 实现使用 OAEP 解密时，先检查 lHash，若失败返回“错误标签”；再检查 0x01，若失败返回“格式错误”。这样的区分对攻击者意味着什么？请利用 oracle 的完备性说明为什么这会导致 Bleichenbacher 攻击变种重新适用。
