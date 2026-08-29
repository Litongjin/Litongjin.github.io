---
title: "每日基础技术总结 · 2026-08-29 · RSA 的 OAEP 填充与 Bleichenbacher 攻击"
date: 2026-08-29 08:00:00
categories: [技术分享]
tags: ["技术分享", "安全基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-29 · RSA 的 OAEP 填充与 Bleichenbacher 攻击

## 📚 今日主题

> **RSA 的 OAEP 填充与 Bleichenbacher 攻击**（安全基础）

### 1. 核心概念速览
RSA的OAEP填充（Optimal Asymmetric Encryption Padding）是将确定性公钥置换（RSA原函数）转换为带随机性的安全加密方案的填充协议。它解决的是'明文可预测性'和'密文可篡改性'问题：原始RSA加密 y = m^e mod n 是确定性的，同一明文永远得到同一密文，且满足乘法同态，无法抵抗选择明文攻击。OAEP在加密前对明文进行Feistel网络变换，混入随机种子并用私钥侧可验证的冗余（标签哈希、0x01分隔符）约束密文的合法性，使得密文的任何扰动在解密时以压倒性概率导致整体校验失败。Bleichenbacher攻击则是针对旧版PKCS#1 v1.5填充的著名攻击：由于v1.5填充的合法结构（0x00 0x02 ... 0x00 M）易被检测，服务器对密文解密后是否合法的响应构成了一个'padding oracle'，攻击者通过大量选择密文查询和区间收缩算法，可以在多项式时间内恢复任意密文对应的明文。OAEP由此成为现代RSA加密标准（PKCS#1 v2.x）的推荐填充方案。该知识处于密码学/公钥加密与可证明安全的交汇点，也是TLS握手（RSA密钥交换）历史漏洞链条的核心环节。专业工程师必须理解它，才能在系统设计中避免误用填充模式、正确评估密码学库的安全边界，并识别类似'oracle'泄露模式。

### 2. 底层原理剖析
一、RSA代数基础
RSA加密 y = x^e mod n 是对整数模n的置换。由于是对乘法群中的元素求幂，它天然同态：对任意s，有 (x^e mod n) * (s^e mod n) ≡ (x*s)^e mod n。因此，攻击者如果要解密c = m^e mod n，可以先选随机数s，计算c' = c * s^e mod n，这样c'对应的明文是m*s mod n。如果服务器解密c'后暴露了关于m*s的某些信息（例如是否符合某种填充格式），攻击者就能反过来约束m的取值。这一机制是所有padding oracle攻击的代数核心，Bleichenbacher是其中最著名的一种。
二、OAEP编码与可证明安全
OAEP使用两个哈希机制：Hash（如SHA-256）和MGF1（Mask Generation Function，以Hash为底层，重复计算Hash(seed || counter)生成任意长度伪随机掩码）。记模长为k字节，Hash长度为hLen。加密前：
1. 生成随机种子r（长度hLen）。
2. lHash = Hash(L)，L是标签（通常为空）。
3. 构造数据块DB = lHash || PS || 0x01 || 消息M，其中PS是零字节填充，使得DB长度k-hLen-1。
4. 用随机种子r生成掩码，遮蔽DB：maskedDB = DB XOR MGF1(r, k-hLen-1)。
5. 再生成掩码遮蔽种子：seedMask = MGF1(maskedDB, hLen)，maskedSeed = r XOR seedMask。
6. 最终EM = 0x00 || maskedSeed || maskedDB，并转为整数进行RSA运算。
解密过程反向执行：先计算RSA逆，然后恢复r和DB，先验证首字节为0，再验证lHash一致，再验证0x01分隔符前都是0。所有验证失败必须返回相同的错误（甚至相同耗时），任何区分都会变成oracle。OAEP在随机预言机模型下被证明是IND-CCA2安全的：即使攻击者选择任意密文，也无法从解密响应中提取关于明文的有效信息。
三、Bleichenbacher攻击算法概要
目标：恢复密文c对应的明文m。假设服务器存在v1.5 oracle，返回TRUE表示解密结果以0x00 0x02开头，FALSE表示不是。攻击者维护关于m的候选区间集合S（初始为[2B, 3B)，其中B=2^(8(k-2))），不断选择整数s，发送c' = c*s^e mod n。若oracle返回TRUE，则说明m*s mod n落在合法填充区间[2B, 3B)，据此可以将S每个区间与m*s mod n ∈ [2B, 3B)的集合取交，逐步缩小候选区间。重复若干轮后，S中只剩一个数，即m。Bleichenbacher原始论文给出约2^20次查询即可恢复明文，在特定TLS实现上可在数分钟内完成，因此该攻击曾对TLS 1.0等造成严重威胁。
四、与前端已有概念的对比
OAEP的'可验证冗余'类似TypeScript接口在编译时的结构约束，但OAEP在运行时强制执行，且对篡改极为敏感。前端常见错误是'不同错误分支返回不同message'，这恰恰就构成oracle。Bleichenbacher攻击本质上是利用布尔型反馈（合法/非法）作为边信道，与前端通过JSON错误码推断数据库结构的场景同构。MGF1的迭代结构类似流密码的密钥流生成，但这里的掩码是公开构造的一部分，其安全性依赖于掩码与数据的互不可分性。

### 3. 基础代码与实战验证
```text
# RSA-OAEP 核心编解码（仅演示格式，真实RSA大数运算省略）
import os, hmac, hashlib

def H(data):
    return hashlib.sha256(data).digest()

def mgf1(seed, length):
    out = b''
    c = 0
    while len(out) < length:
        out += H(seed + c.to_bytes(4, 'big'))   # 掩码流：Hash(seed || counter)
        c += 1
    return out[:length]

def xor(a, b):
    return bytes(x ^ y for x, y in zip(a, b))

def constant_time_eq(a, b):
    return hmac.compare_digest(a, b)  # 恒时比较，防止时序侧信道

def oaep_encode(m, k, label=b''):
    h_len = len(H(b''))
    assert len(m) <= k - 2*h_len - 2
    r = os.urandom(h_len)            # 随机种子，使每次加密结果不同
    l_hash = H(label)                # 标签哈希，固定上下文的校验值
    ps_len = k - len(m) - 2*h_len - 2
    db = l_hash + b'\x00' * ps_len + b'\x01' + m  # 0x01是明文的起始标记
    masked_db = xor(db, mgf1(r, k - h_len - 1))
    masked_seed = xor(r, mgf1(masked_db, h_len))
    return b'\x00' + masked_seed + masked_db

def oaep_decode(em, label=b''):
    k = len(em)
    if em[0] != 0:                  # 首字节必须为0，作为整体完整性检查之一
        return None
    h_len = len(H(b''))
    masked_seed = em[1:1+h_len]
    masked_db = em[1+h_len:]
    r = xor(masked_seed, mgf1(masked_db, h_len))
    db = xor(masked_db, mgf1(r, len(masked_db)))
    if not constant_time_eq(db[:h_len], H(label)):  # 标签校验，错误时返回统一值
        return None
    i = db.find(b'\x01', h_len)
    if i < 0:                       # 找不到分隔符，说明结构被破坏
        return None
    if any(x != 0 for x in db[h_len:i]):  # 0x01之前必须是全零PS
        return None
    return db[i+1:]                 # 提取真实明文

# Bleichenbacher attack 的 oracle 示意：
# for s in range(1, B):
#     if oracle(c * pow(s, e, n) % n):
#         # 合法填充 -> 收紧 m 的有效区间
#         update_bounds(s)
# 这个循环配合区间相交算法，最终唯一确定 m。
```

### 4. 常见误区与进阶思考
常见误区：
1. 认为'填充只是为了保证长度'。实际上OAEP填充的核心不是让明文长度凑满模长，而是创建一种'带公钥可验证的冗余结构'，将RSA的确定性置换转变为带随机且可验证的加密方案。仅仅用传统padding（如PKCS#1 v1.5）虽然长度合法，但缺少可证明安全性，会直接被Bleichenbacher oracle打穿。
2. 认为'只要统一错误信息就安全'。忽略侧信道攻击者可以通过响应耗时、内存访问模式、CPU功耗等区分不同失败路径。OAEP实现中必须使用恒时比较、恒定流程和统一返回，否则时序差异本身就是可用的oracle。
思考题：假设服务器实现了RSA-OAEP解密，并对所有解密失败返回同一个布尔值，但在处理'首字节非0'时耗时极短（提前返回），而处理'标签校验失败'时进行了恒时但更长的运算。请说明攻击者如何利用这个可观测的时间差构造一个oracle，并设计一个最小化的Bleichenbacher式语义恢复攻击来破解一个已知密文？（提示：时间差将'首字节校验'这一内部状态开放为外部信号。）
