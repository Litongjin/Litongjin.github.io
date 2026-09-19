---
title: "每日基础技术总结 · 2026-09-19 · RSA 的 OAEP 填充与 Bleichenbacher 攻击"
date: 2026-09-19 08:00:00
categories: [技术分享]
tags: ["技术分享", "安全基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-19 · RSA 的 OAEP 填充与 Bleichenbacher 攻击

## 📚 今日主题

> **RSA 的 OAEP 填充与 Bleichenbacher 攻击**（安全基础）

### 1. 核心概念速览
**定义**。RSA 本身只是一个确定性陷门置换（TDP）：`E(m)=m^e mod n`，`D(c)=c^d mod n`，输入输出都在 `Z_n` 上、长度等于模数字节数 `k=ceil(bitlen(n)/8)`。它既不随机、也不可证明不可区分，更无完整性。所谓「填充」不是对齐或凑长度，而是把 TDP 升级为 IND-CCA2 安全公钥加密方案的必要构件：在明文前构造一段有冗余、且依赖随机数的编码 EM（Encoded Message），再做 `c = EM^e mod n`。

**两大填充标准**（PKCS#1 / RFC 8017）：
- `RSAES-PKCS1-v1_5`（v1.5，1993）：`EM = 0x00 || 0x02 || PS || 0x00 || M`，PS 为 ≥8 字节非零随机串。
- `RSAES-OAEP`（1994 Bellare–Rogaway，v2.x）：`EM = 0x00 || maskedSeed || maskedDB`，`DB = lHash || PS(0x00…) || 0x01 || M`，两层 MGF1 掩码构成 Feistel 式结构；在随机预言机模型下，若 RSA 是 TDP 则 OAEP 达到 IND-CCA2。

**Bleichenbacher 攻击（CRYPTO'98）**。它针对 v1.5 加密填充：只要攻击者能区分「解密后 padding 结构合法」与「非法」（一个 1-bit 判定 oracle），就能结合 RSA 的乘法同态性 `E(a·b) = E(a)·E(b) mod n` 做选择密文攻击，通过自适应查询把明文 `m` 的候选区间从 `[2B, 3B)`（`B = 2^{8(k-2)}`）逐步收缩到单点，1024-bit 模数下约 2^20 次查询即可完整恢复明文，无需分解 n。后续 Manger(2001) 用首字节是否为 `0x00` 的判定位击穿了 OAEP 的朴素实现；ROBOT(2017) 证明该漏洞在大量 TLS 栈中因实现细节（提前 return、错误码不同、时序差异）反复复活。

**体系位置**。它处在「公钥原语 → 填充方案 → 协议握手 → 应用层密钥管理」这条链的正中间：TLS 1.0–1.2 的 RSA 密钥传输、S/MIME、XML Encryption、JWT 的 `RSA-OAEP`、Java `Cipher.getInstance("RSA/ECB/PKCS1Padding")` 全都踩在这条线上。TLS 1.3 直接删除了 RSA 密钥传输（只保留 RSA-PSS 签名 + (EC)DHE），就是对这一族攻击的工程性终结。对后端/AI 工程师而言，这是一条通用纪律的源头：**任何依赖秘密数据、且其成功/失败可被外部观测的分支，都是一个 oracle**。

### 2. 底层原理剖析
**一、v1.5 编码与合法区间**
把 `EM` 按大端解释为整数 `m`。由于 `EM[0]=0x00, EM[1]=0x02`，必然有 `2·B ≤ m < 3·B`，其中 `B = 2^{8(k-2)}`。反过来，落在这个区间内的整数**不**一定结构合法（还要校验 PS 非零、存在 0x00 分隔符、PS 长度 ≥ 8），但区间给出了一个极强的一阶约束——攻击正是从这个约束出发的。

**二、攻击骨架（Bleichenbacher 1998）**
记目标密文 `c0`，其明文 `m0 ∈ [2B, 3B)`（先做盲化保证这一点）。攻击者自选整数 s，构造 `c' = c0 · s^e mod n`，提交给解密 oracle。因为 `c' = (m0·s)^e mod n`，oracle 返回「合法」当且仅当

    2B ≤ (m0 · s mod n) < 3B

即存在整数 r 使 `2B + r·n ≤ m0·s < 3B + r·n`，等价于

    m0 ∈ ⋃_r [ (2B + r·n)/s , (3B - 1 + r·n)/s ]

于是候选集合 `M`（一个区间的集合）被 `M ← M ∩ 上述并集` 收窄。每轮命中一次，`M` 的总测度乘以约 `B/n`；迭代约 `log(n)/log(n/B) = 8k/16` 轮后 `M` 收缩为单点。

**三、s 的选取（决定查询复杂度）**
- 首轮：`s1 = ceil(n / 3B)`（这是使 `m0·s1 mod n` 有机会落回区间的下界）。
- `|M| > 1`：`s ← s+1` 线性试。
- `|M| = 1 = [lo, hi]`：不再线性，解析跳跃——取满足 `s·hi ≥ 2B + r·n` 的最小 `r`，再令 `s = max(s+1, ceil((2B + r·n)/hi))`，并校验 `lo·s ≤ 3B - 1 + r·n`（保证交集非空）。这一段的存在是查询数落到 2^20 而非爆炸的关键。

**四、OAEP 的构造与它的「另一条缝」**
加密：

    lHash   = Hash(L)
    DB      = lHash || 0x00^(k-|M|-2hLen-2) || 0x01 || M      # 长度 k-hLen-1
    seed    = random(hLen)
    maskedDB   = DB   XOR MGF1(seed,     k-hLen-1)
    maskedSeed = seed XOR MGF1(maskedDB, hLen)
    EM      = 0x00 || maskedSeed || maskedDB

解密必须**反序**去掩码（`seed` 依赖 `maskedDB`，`DB` 依赖 `seed`）。OAEP 把「结构冗余」换成了「哈希冗余」，并把校验推迟到解码之后，因此对 Bleichenbacher 免疫。但 `EM[0]` 恒为 `0x00` 这一事实本身就是可判定的信息：`if em[0] != 0: raise` 这条早退分支，让攻击者获得「`m·s mod n` 是否 < 2^{8(k-1)}」的判定位，配合同样的乘法同态性即可复刻区间收缩——这就是 Manger 攻击。

**五、与前端概念的对照**

| 维度 | 前端已有的概念 | RSA 填充 / CCA oracle |
| --- | --- | --- |
| 校验时机 | TypeScript 类型守卫（`x is T`）只在编译期存在，运行时被擦除 | padding/hash 校验是运行时的字节级结构判定，构成解码路径上的真实分支 |
| 失败反馈 | `JSON.parse` 抛 SyntaxError；接口用 401/404/422 区分语义 | 这个「区分」本身就是攻击面：解密失败的反馈 = CCA oracle |
| 纯函数与可组合性 | 纯函数 `f` 满足 `f(a·b)=f(a)·f(b)` 是优点（可缓存、可组合、易于测试） | RSA 的乘法同态 `E(a·b)=E(a)E(b)` 正好是这个性质，而它就是攻击的杠杆 |
| 时间/状态一致性 | 前端不关心（渲染耗时不影响安全边界） | 密钥材料路径上任何数据相关的分支、比较、内存访问模式都是漏洞 |

核心差异一句话：前端的类型/校验边界是「正确性边界」，密码学的解码边界是「安全性边界」，前者的失败可以随便报，后者的失败必须**不可区分**。

### 3. 基础代码与实战验证
```text
以下脚本只依赖 Python 标准库，同时演示 v1.5 填充、Bleichenbacher 完整恢复，以及 OAEP 的结构对照。

import os, math, random, hashlib

# ---------- 0. 最小 RSA 原语 ----------
def _mr(n, a):                       # Miller-Rabin 单轮
    d, s = n - 1, 0
    while d % 2 == 0:
        d //= 2
        s += 1
    x = pow(a, d, n)
    if x == 1 or x == n - 1:
        return True
    for _ in range(s - 1):
        x = x * x % n
        if x == n - 1:
            return True
    return False

def gen_prime(bits):
    while True:
        p = random.getrandbits(bits) | (1 << (bits - 1)) | 1   # 强制最高位与最低位为 1
        if all(_mr(p, a) for a in (2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37)):
            return p

BITS, E = 640, 65537                 # 玩具参数；生产环境 n >= 2048 bit
while True:
    p, q = gen_prime(BITS // 2), gen_prime(BITS // 2)
    if p == q:
        continue
    n = p * q
    phi = (p - 1) * (q - 1)
    if math.gcd(E, phi) == 1:
        D = pow(E, -1, phi)          # 私钥指数 = e 关于 phi 的模逆
        break
k = (n.bit_length() + 7) // 8        # 模数字节长度；EM 必须恰好等长（I2OSP 定长语义）

# ---------- 1. PKCS#1 v1.5 加密填充 ----------
def pad_v15(msg):
    # EM = 0x00 || 0x02 || PS(>=8 字节非零随机) || 0x00 || M
    assert len(msg) <= k - 11
    ps = bytes(x or 1 for x in os.urandom(k - 3 - len(msg)))   # 0 -> 1，保证 PS 全非零
    return bytes([0, 2]) + ps + bytes([0]) + msg

def unpad_v15(em):
    if len(em) != k or em[0] != 0 or em[1] != 2:
        return None
    i = em.find(0, 2)                # 定位 PS 与 M 之间的 0x00 分隔符
    if i < 10:                       # PS 长度 = i-2 >= 8  =>  i >= 10
        return None
    return em[i + 1:]

def enc_v15(msg):
    return pow(int.from_bytes(pad_v15(msg), 'big'), E, n)

def oracle_v15(c):
    # 服务端视角。攻击者只能观测到这个布尔值 —— 这就是全部的信息泄漏面
    em = pow(c, D, n).to_bytes(k, 'big')      # I2OSP：定长 k 字节，高位自动补 0
    return unpad_v15(em) is not None

# ---------- 2. Bleichenbacher 选择密文攻击 ----------
def cdiv(a, b):
    return -(-a // b)                # 向上取整，对负数同样成立

def attack(c0, verbose=True):
    B = 1 << (8 * (k - 2))           # 2B <= m < 3B  <==>  EM 以 0x00 0x02 开头
    M = [(2 * B, 3 * B - 1)]         # 候选区间集合
    s = cdiv(n, 3 * B)               # s1 = ceil(n / 3B)
    q = 0
    while True:
        # 2a. 找到最小的 s，使 c0 * s^e mod n 被 oracle 判定为合法
        while True:
            q += 1
            if oracle_v15(c0 * pow(s, E, n) % n):
                break
            s += 1
        # 2b. 用 m0*s 落在 [2B, 3B) 这一约束收窄所有候选区间
        newM = []
        for lo, hi in M:
            for r in range(cdiv(lo * s - 3 * B + 1, n), (hi * s - 2 * B) // n + 1):
                a = max(lo, cdiv(2 * B + r * n, s))
                b = min(hi, (3 * B - 1 + r * n) // s)
                if a <= b:
                    newM.append((a, b))
        M = newM
        if verbose:
            print(f'oracle={q} 区间数={len(M)} 宽度={M[0][1] - M[0][0]}')
        if len(M) == 1 and M[0][0] == M[0][1]:
            return M[0][0], q       # 区间塌缩为单点 == 明文恢复
        # 2c. 选择下一个 s
        if len(M) == 1:
            lo, hi = M[0]
            r = (hi * s - 2 * B) // n + 1        # 让 s 必须前进的最小 r
            while True:
                s = max(s + 1, cdiv(2 * B + r * n, hi))
                if lo * s <= 3 * B - 1 + r * n:  # 交集非空
                    break
                r += 1
        else:
            s += 1

msg = b'TOP'
c0 = enc_v15(msg)
m0_true = pow(c0, D, n)
assert oracle_v15(c0)                # 前提：目标密文本身是结构合法的
m0, queries = attack(c0)
print('恢复正确 =', m0 == m0_true, '总 oracle 调用 =', queries)

# ---------- 3. 对照：RSA-OAEP (PKCS#1 v2.2) ----------
HL = 32                              # SHA-256，要求 k >= 2*HL + 2 = 66

def mgf1(seed, length, h=hashlib.sha256):
    out = b''
    for i in range(cdiv(length, h().digest_size)):
        out += h(seed + i.to_bytes(4, 'big')).digest()   # 计数器 4 字节大端
    return out[:length]

def oaep_encrypt(msg, label=b''):
    if len(msg) > k - 2 * HL - 2:
        raise ValueError('message too long')
    lhash = hashlib.sha256(label).digest()
    db = lhash + bytes(k - len(msg) - 2 * HL - 2) + bytes([1]) + msg   # 长度 k-HL-1
    seed = os.urandom(HL)
    maskedDB = bytes(x ^ y for x, y in zip(db, mgf1(seed, k - HL - 1)))
    maskedSeed = bytes(x ^ y for x, y in zip(seed, mgf1(maskedDB, HL)))
    em = bytes([0]) + maskedSeed + maskedDB    # 首字节恒为 0x00，解密时的第一个检查点
    return pow(int.from_bytes(em, 'big'), E, n)

def oaep_decrypt(c):
    em = pow(c, D, n).to_bytes(k, 'big')
    if em[0] != 0:                             # ← Manger 攻击利用的正是这次早退
        raise ValueError('decryption error')
    maskedSeed, maskedDB = em[1:1 + HL], em[1 + HL:]
    seed = bytes(x ^ y for x, y in zip(maskedSeed, mgf1(maskedDB, HL)))
    db = bytes(x ^ y for x, y in zip(maskedDB, mgf1(seed, k - HL - 1)))
    if db[:HL] != hashlib.sha256(b'').digest():
        raise ValueError('decryption error')   # 必须与上一条异常完全不可区分
    i = db.find(1, HL)
    if i < 0:
        raise ValueError('decryption error')
    return db[i + 1:]

assert oaep_decrypt(oaep_encrypt(b'secret')) == b'secret'

关键行说明：`unpad_v15` 的布尔返回就是 oracle 的全部输出；`c0 * pow(s, E, n) % n` 是利用乘法同态在不接触私钥的前提下构造等价密文；`M[0][0] == M[0][1]` 是收敛判据；OAEP 里两处 `raise` 位置不同——只要存在可区分的差异（异常类型、消息长度、分支耗时），IND-CCA2 的安全证明就不再适用。
```

### 4. 常见误区与进阶思考
**误区一：把「解密失败原因统一成一个 error message」当成已经修好**。统一文案只堵住了一类信道（错误文本），而 oracle 是**任何**与秘密数值相关的可观测差异：`if em[0] != 0: raise` 的提前返回、`memcmp` 在首个不等字节处返回的时序差、异常构造本身的开销差，都会重新打开判定位。ROBOT(2017) 之所以能一次扫出大量存在十余年的实现缺陷，正是因为它用网络层可测量的时序与行为差异重建了 Bleichenbacher oracle。工程上的正确做法只有两条：(a) 常数时间实现 —— 对 padding/hash 用累积 flag 而非早退分支，比较全程遍历，错误出口唯一；(b) 干脆不提供这个 oracle —— 改用 RSA-KEM（随机 KEM 密钥 + KDF）或 ECDH/HPKE 做密钥封装，这正是 TLS 1.3 删除 RSA 密钥传输的原因。补一句：光有常数时间还不够，还必须「先验后解」地穷尽校验路径——例如先查首字节、再查 hash、再查 0x01 分隔符，三步的错误出口必须在同一条路径上对齐。

**误区二：把它当成「PKCS#1 v1.5 是旧版、OAEP 是新版，可以互换/升级」**。二者是完全不兼容的编码格式，不能互操作；OAEP 的明文上限是 `k - 2·hLen - 2`（2048-bit + SHA-256 只有 190 字节），比 v1.5 的 `k - 11`（245 字节）更小；OAEP 只提供机密性、不提供完整性，所以实践中它是「RSA-OAEP 封装一个 AES 会话密钥，再用 AEAD 加密数据」的混合加密（RSA 存在 n、e、d 也只是数学上的模型，真正的数据加密永远走对称原语）。另一个同源误区是把填充理解成「凑长度/字节对齐」（从 AES-CBC 的 PKCS#7 padding 带过来的直觉）——RSA 填充从一开始就是安全机制：随机化提供语义安全所需的不可区分性，结构冗余提供解密时的合法性判定，两者共同把 TDP 变成加密方案。

**误区三（附带）**：认为「明文一样，密文一样」是性能优势（可以类比 HTTP 缓存或前端 memoization）。确定性恰恰是可区分性的同义词，而 RSA 的同态性又让密文之间可以做乘法运算——这是密码学里代数结构既是武器也是攻击面的典型例证。

**思考题**：假设服务端做到以下几点——OAEP 解密的失败一律抛同一个异常、HTTP 状态码固定 400、响应体完全一致，但代码里 `em[0] != 0` 时立即 return，而 `db[:HL] != lHash` 时会在两次 MGF1 展开之后才 return。请回答：(1) 攻击者是否仍然能够构造出有效的 oracle？如果能，他需要观测什么（响应时间？TCP 分段？连接关闭方式？），以及需要多少样本才能把两个分支区分开来？(2) 进一步假设服务端连时序都严格常数化，但允许攻击者反复提交同一密文——那么密文的**可重放性**、**连接复用**、**错误重试计数**这类协议层行为是否又构成了新的信道？(3) 最后，回到设计层面：为什么现代协议（TLS 1.3、HPKE）选择直接删掉 RSA 解密这条路径，而不是继续在实现层面「打补丁」？请从「可证明安全需要理想化模型」与「实现暴露的接口远多于模型假设」这两个角度回答。
