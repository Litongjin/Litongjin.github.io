---
title: "每日基础技术总结 · 2026-08-24 · RSA 的 OAEP 填充与 Bleichenbacher 攻击"
date: 2026-08-24 08:00:00
categories: [技术分享]
tags: ["技术分享", "安全基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-24 · RSA 的 OAEP 填充与 Bleichenbacher 攻击

## 📚 今日主题

> **RSA 的 OAEP 填充与 Bleichenbacher 攻击**（安全基础）

### 1. 核心概念速览
OAEP（Optimal Asymmetric Encryption Padding，最优非对称加密填充）是 RSA 加密中用于将确定性算法转化为概率性算法的填充方案，本质是在加密前对明文进行可逆的随机化编码，使得同一明文多次加密产生不同密文，并引入解密时的完整性校验。它解决的是 RSA 裸加密（教科书式 RSA）的两个根本缺陷：确定性导致密文可被猜测和字典攻击；以及代数结构导致密文可被篡改和同态运算攻击。Bleichenbacher 攻击是针对 RSA-PKCS#1 v1.5 填充的一种选择密文攻击（CCA），利用解密端对填充格式校验失败与成功的行为差异（即 padding oracle），通过反复提交经过变换的密文，逐步恢复明文。该知识点属于公钥密码学、加密协议设计和侧信道攻击的交叉领域，是理解 TLS 1.2 及之前版本 RSA 密钥交换中若干已知漏洞的基石。专业工程师必须掌握它，因为任何涉及 RSA 加密的现代系统（如 TLS、JWT、OpenPGP）都必须正确处理填充与错误响应，否则即使算法数学上安全，协议层也会被攻破。理解这一原理有助于建立『密码学安全不等于算法安全，而是协议与实现共同安全』的底层认知。

### 2. 底层原理剖析
OAEP 的底层机制基于 Feistel 网络结构的可逆变换。设 RSA 模数为 n，k 为 n 的字节长度。OAEP 输入明文 m 长度最多为 k - 2 - 2hLen（hLen 为哈希函数输出长度）。填充过程：1) 生成随机数 r（长度为 hLen）；2) 将 m 左侧填充 0 和固定字节 0x01，得到数据块 DB = 0x00...01 || m；3) 计算 lHash = Hash(标签)，通常标签为空；4) 构造 DB = lHash || PS || 0x01 || m，其中 PS 为若干 0x00；5) 使用掩码生成函数 MGF 对 r 进行扩展得到掩码 maskedDB = DB XOR MGF(r)；6) 计算 maskedR = r XOR MGF(maskedDB)；7) 最终编码为 maskedR || maskedDB，再通过 RSA 加密（模幂运算）得到密文。解密时反向操作，先计算 maskedR 和 maskedDB，再恢复 r 和 DB，校验 lHash 和 0x01 分隔符，全部通过则输出 m，否则报错。关键点：任何一步校验失败，解密端必须以相同错误响应，否则形成 padding oracle。Bleichenbacher 攻击利用的是 PKCS#1 v1.5 填充的简单结构：加密后的明文块以 0x00 0x02 开头，随后是非零随机填充，然后 0x00 分隔符后是明文。攻击者截获密文 c 后，对任意整数 s，计算 c' = c * s^e mod n，并提交给解密端。解密端若报告『填充正确』（即解密结果前两字节为 0x00 0x02），则攻击者知道 c' 对应的明文位于特定区间；反复尝试不同 s，通过二分法缩小明文可能区间，最终恢复原明文 m。攻击复杂度约为 2^20 次查询，对于 1024 位 RSA 在早期实验中可以数分钟内完成。对比前端已有概念：OAEP 中的随机化类似于 JavaScript 中 `crypto.getRandomValues` 为密码学操作注入熵，但关键区别在于前端通常关注随机数质量，而 OAEP 关注的是将随机性嵌入编码结构并保证可逆性；Bleichenbacher 攻击则类似于前端中的『时序攻击』（如比较字符串时使用 `===` 导致时序差异），本质都是利用行为可观测差异（oracle）来推断秘密信息，但前者是数学区间泄露，后者是执行时间泄露。与 TypeScript/Java 接口对比：接口是编译期约束，属于类型系统，而 OAEP 是运行期数据变换，属于编码方案，两者解决不同层面的『契约』问题——接口契约防止类型错误，OAEP 契约防止密文被篡改。

### 3. 基础代码与实战验证
由于 OAEP 和 Bleichenbacher 攻击涉及复杂数学运算，以下给出核心逻辑的精确伪代码步骤，不依赖任何框架。

1. OAEP 加密（RSA-OAEP）:
```
function OAEP_Encrypt(message, rsa_e, rsa_n, hash, mgf):
    k = byteLength(rsa_n)          // RSA 模数的字节长度
    hLen = hash.outputLength        // 例如 SHA-256 为 32 字节
    maxMsgLen = k - 2 * hLen - 2    // 明文最大长度
    assert length(message) <= maxMsgLen
    r = randomBytes(hLen)           // 生成随机种子，这是概率性的来源
    lHash = hash('')                // 空标签的哈希
    PS = zeros(maxMsgLen - length(message) - 1)  // 填充零
    DB = lHash || PS || 0x01 || message
    seedMask = mgf(r, k - hLen - 1) // 用 MGF 扩展 r 生成掩码
    maskedDB = DB XOR seedMask      // 掩盖数据块
    dbMask = mgf(maskedDB, hLen)    // 用 maskedDB 生成另一个掩码
    maskedR = r XOR dbMask          // 掩盖随机种子
    encoded = 0x00 || maskedR || maskedDB   // 最终 EM 编码
    c = modularExponentiation(encoded, rsa_e, rsa_n)  // RSA 模幂
    return c
```

2. OAEP 解密（解密端必须注意错误统一）:
```
function OAEP_Decrypt(c, rsa_d, rsa_n, hash, mgf):
    encoded = modularExponentiation(c, rsa_d, rsa_n)  // 还原编码块
    if byteLength(encoded) != k: return ERROR
    if encoded[0] != 0x00: return ERROR
    maskedR = encoded[1 : hLen+1]
    maskedDB = encoded[hLen+1 : ]
    dbMask = mgf(maskedDB, hLen)
    r = maskedR XOR dbMask
    seedMask = mgf(r, k - hLen - 1)
    DB = maskedDB XOR seedMask
    lHash' = DB[0 : hLen]
    if lHash' != hash(''): return ERROR
    // 从 DB 中定位 0x01 分隔符，且之前必须全部是 0x00
    find index i > hLen such that DB[i] == 0x01 and all DB[hLen..i-1] == 0x00
    if not found: return ERROR
    message = DB[i+1 : ]
    return message
```

3. Bleichenbacher 攻击核心（演示攻击者视角）:
```
function Bleichenbacher_Attack(ciphertext, oracle, rsa_e, rsa_n):
    // oracle(c) 返回解密后是否满足 PKCS#1 v1.5 填充格式（前两字节 0x00 0x02）
    B = 2^(8*(k-2))                 // 有效填充区间的下界
    M = [[2*B, 3*B - 1]]            // 初始明文区间（满足 0x00 0x02 开头的区间）
    s = 1
    while true:
        // 寻找 s 使得 c' = c * s^e mod n 通过 oracle
        s = find_next_s(s, ciphertext, oracle, rsa_e, rsa_n)
        // 利用区间交集公式缩小 M，直到只含一个整数
        M = intersect_intervals(M, s, ciphertext, oracle, rsa_e, rsa_n)
        if len(M) == 1:
            m = M[0]
            // 验证 m^e mod n == ciphertext
            return m
```

关键注释：第 1 步中的 `modularExponentiation` 是 RSA 核心数学运算，即计算 `base^exp mod n`；第 2 步中所有 `return ERROR` 必须使用相同错误消息和相同执行时间（或引入随机延时），否则就产生 oracle；第 3 步中 `oracle` 就是解密端对填充校验的响应差异，攻击的本质是利用这个差异做数学区间收敛。实际工程中必须使用标准库的 `RSA-OAEP` 而非手写，因为任何微小实现偏差都会导致漏洞。

### 4. 常见误区与进阶思考
误区一：认为 RSA 算法本身安全就足够，忽略填充与错误处理的实现细节。Bleichenbacher 攻击恰恰证明，即使 RSA 数学难题未被破解，只要解密端对填充校验失败和成功的响应有可观测差异，攻击者就能完整恢复明文。这在实践中表现为：开发者自实现 RSA 解密后返回不同的异常类型、HTTP 状态码或日志信息，或使用非恒定时间比较，都会将系统变成 oracle。前端工程师容易类比为：以为用了 HTTPS 就安全，却忽略 TLS 库版本或证书校验逻辑。必须认识到密码学协议安全是整体的，任何旁路信息都是攻击面。

误区二：误以为 OAEP 能防止所有选择密文攻击。OAEP 本身被证明在随机预言机模型下是 IND-CCA2 安全的，但前提是哈希函数和 MGF 实现正确，且解密过程对错误处理是原子化的。如果使用不正确的 MGF（如将 MGF1 与不同哈希混用）、标签不一致、或者对错误信息分类返回，OAEP 的保护也会失效。此外，Bleichenbacher 攻击是针对 PKCS#1 v1.5 的，虽然现代 TLS 已改用 RSA-OAEP 或 ECDHE，但遗留系统中仍有大量 PKCS#1 v1.5 实现。前端工程师可能只关注加密算法的强度，却忽略协议版本协商、填充模式等元数据，这是不够的。

思考题：假设你负责设计一个基于 RSA 的加密 API，你决定使用 OAEP 填充，但为了调试方便，在解密失败时返回 `'DECRYPT_FAILED'` 错误码，成功时返回明文。请问这种设计在什么条件下会引入可被利用的 oracle？为什么即使攻击者无法直接调用解密 API，网络时间测量也可能构成 oracle？请从 Bleichenbacher 攻击的查询模型与侧信道角度分析。
