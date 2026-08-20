---
title: "每日基础技术总结 · 2026-05-26 · RSA 的 OAEP 填充与 Bleichenbacher 攻击"
date: 2026-05-26 08:00:00
categories: [技术分享]
tags: ["技术分享", "安全基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-05-26 · RSA 的 OAEP 填充与 Bleichenbacher 攻击

## 📚 今日主题

> **RSA 的 OAEP 填充与 Bleichenbacher 攻击**（安全基础）

### 1. 核心概念速览
RSA-OAEP 是一种将确定性 RSA 加密转化为概率性加密的填充方案，其本质是在加密前通过 Feistel 网络将明文与随机种子混合，使得同一明文在多次加密中产生不同密文，并在解密后通过校验结构检测篡改或错误。它解决的核心问题是 RSA 原始操作（c = m^e mod n）缺乏语义安全性，且简单填充（如 PKCS#1 v1.5）存在 Bleichenbacher 攻击的漏洞。Bleichenbacher 攻击是一种针对 RSA 填充的 padding oracle 攻击：攻击者利用服务器对密文解密后填充是否合法（如是否以 0x00 0x02 开头）的响应差异，借助 RSA 的同态性质反复修改密文，逐步收窄明文区间，最终在约 2^20 次查询内恢复明文。该知识属于公钥密码学实践安全的核心范畴，是理解 padding oracle、TLS 协议安全演进（如 Lucky13）及可证明安全的基础。专业工程师必须掌握，因为任何可区分性 oracle 都可能成为攻击面，设计协议时必须假设错误信息会被利用。

### 2. 底层原理剖析
OAEP 的底层机制：加密时，设明文为 m，填充后得到 m' = m || 0^k1（其中 k1 为要追加的零比特数）。选取随机种子 r，使用两个哈希函数 G 和 H，构造 Feistel 结构：X = (m' ⊕ G(r))，Y = r ⊕ H(X)。密文为 RSA 加密 (X || Y)。解密时，先计算 X,Y，再恢复 r = Y ⊕ H(X)，然后 m' = X ⊕ G(r)，最后检查 m' 的低 k1 位是否全零，若否则拒绝。该结构在随机预言机模型下可证明是 IND-CPA 安全的。

Bleichenbacher 攻击的数学原理：PKCS#1 v1.5 填充格式为 0x00 0x02 || PS || 0x00 || M，其中 PS 为至少 8 个非零随机字节。攻击者截获密文 c = m^e mod n，选择因子 s，计算 c' = c * s^e mod n = (m*s)^e mod n。服务器解密得到 m' = m*s mod n，并检查其是否满足填充格式。攻击者根据响应是“有效”还是“无效”得到关于 m*s mod n 是否落在合法填充区间（即前两字节为 0x00 0x02，且后跟至少 8 个非零字节）的布尔信息。通过调整 s，该信息将 m 的取值空间逐步分割，类似二分搜索，最终唯一确定 m。与前端概念对比：这类似于前端通过 API 返回的 HTTP 状态码差异（如 401 与 403）来推断用户是否存在，本质都是利用可区分响应泄露内部状态。而 OAEP 类似于给数据加上运行时校验（如 TypeScript 的运行时类型守卫），使得即使密文被篡改，解密后的校验也会失败，且不暴露任何可区分的细节。

### 3. 基础代码与实战验证
```text
以下用 Python 演示 OAEP 加密解密及 Bleichenbacher 攻击核心步骤（仅概念验证，使用 cryptography 库）。

from cryptography.hazmat.primitives.asymmetric import rsa, padding
from cryptography.hazmat.primitives import hashes

# 生成 RSA 密钥
private_key = rsa.generate_private_key(public_exponent=65537, key_size=2048)
public_key = private_key.public_key()

# OAEP 加密：加密时内部会生成随机种子，并执行 Feistel 混合
message = b"RSA OAEP"
ciphertext = public_key.encrypt(
    message,
    padding.OAEP(mgf=padding.MGF1(algorithm=hashes.SHA256()),
                  algorithm=hashes.SHA256(),
                  label=None)
)

# OAEP 解密：解密后检查填充结构，若被篡改则抛出异常
plaintext = private_key.decrypt(
    ciphertext,
    padding.OAEP(mgf=padding.MGF1(algorithm=hashes.SHA256()),
                  algorithm=hashes.SHA256(),
                  label=None)
)
assert plaintext == message

# --- Bleichenbacher 攻击模拟（核心步骤） ---
# 假设 oracle(c) 返回 c^d mod n 是否满足 PKCS#1 v1.5 填充（前两字节为 0x00 0x02）
def oracle(c, private_key):
    # 实际中服务器会执行解密并返回错误码，这里模拟
    m = private_key.decrypt(c)  # 使用 PKCS1v15 解密，但这里为演示直接取原始值
    # 注意：实际攻击中服务器使用 PKCS#1 v1.5 解密，并可能返回 padding 错误
    return (m.bit_length() >= 8*key_size - 8)  # 简化判断

# 攻击者已知 c，选取 s，计算 c' = c * s^e mod n
# 根据 oracle(c') 的结果，判断 m*s mod n 的区间
# 逐步缩小 m 的取值范围，类似二分搜索（具体算法省略）
# 核心同态性质：RSA 乘法同态
s = 2  # 实际会自适应调整
c_prime = (ciphertext * pow(s, 65537, private_key.public_key().public_numbers().n)) % private_key.public_key().public_numbers().n
# 调用 oracle 并收集信息，反复迭代恢复明文。

注意：完整攻击需要数百次查询，此处仅展示同态操作与 oracle 利用的关键点。
```

### 4. 常见误区与进阶思考
常见误区一：认为 OAEP 只是简单加盐。实际上 OAEP 的 Feistel 结构和双向哈希混合提供了可证明的语义安全性，仅加盐（如随机前缀）无法防止篡改或保证安全。
常见误区二：认为 Bleichenbacher 攻击只影响古老的 TLS 1.0。事实上，任何暴露“填充是否合法”这一位信息的系统都可能遭受类似攻击，例如 CBC 模式的 Lucky13 攻击，或某些实现中时序侧信道。即便 TLS 1.3 移除 PKCS#1 v1.5，其他协议或库中的兼容模式仍可能引入漏洞。
思考题：假设有一个 oracle 函数 O(c) = 1 当且仅当 c^d mod n < n/2（即 RSA 私钥操作结果的最高位为 0），其他输入返回 0。给定密文 c，如何利用该 oracle 恢复明文 m？（提示：利用 RSA 同态性质，构造 c' = c * 2^e mod n，则解密结果为 2m mod n，观察其与 n/2 的关系，可逐位恢复 m 的二进制表示。这检验你是否真正理解 RSA 代数结构和信息泄露的本质。）
