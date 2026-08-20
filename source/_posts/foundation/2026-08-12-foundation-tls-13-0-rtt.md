---
title: "每日基础技术总结 · 2026-08-12 · TLS 1.3 握手流程与 0-RTT"
date: 2026-08-12 08:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-08-12 · TLS 1.3 握手流程与 0-RTT

## 📚 今日主题

> **TLS 1.3 握手流程与 0-RTT**（网络基础）

### 1. 核心概念速览
TLS 1.3 (RFC 8446) 是传输层安全协议的最新版本。其握手的本质是一个基于(EC)DHE与PSK的认证密钥协商协议，目标是在一个RTT内完成加密参数协商、身份验证与密钥派生，并支持0-RTT数据发送。它解决的问题是：在不可信网络中安全地建立双向加密信道，同时提供前向保密与抗降级攻击。相比TLS 1.2，它移除了静态RSA、CBC等脆弱机制，强制使用现代AEAD密码套件。在体系位置中，TLS位于TCP之上、应用层之下，是HTTP/2、HTTP/3、gRPC等协议的安全基座。专业工程师必须掌握它，因为性能优化（连接复用、0-RTT）、安全配置（密码套件选择、证书验证）和问题诊断（握手延迟、TLS错误）都建立在对握手流程的精确理解之上。

### 2. 底层原理剖析
完全握手（1-RTT）流程：
1. ClientHello：客户端生成随机数client_random，携带支持的密码套件列表、客户端支持的key_share（临时公钥），若可恢复则携带PSK标识。
2. ServerHello：服务器选择密码套件与自己的key_share，发送server_random。此后所有消息使用握手密钥加密：EncryptedExtensions（扩展）、Certificate（证书链）、CertificateVerify（签名覆盖所有握手消息）、Finished（MAC）。
3. 客户端验证证书与签名，计算共享密钥，发送Finished。然后切换至应用数据密钥。

密钥派生基于HKDF-Extract/Expand，层次如下：
early_secret = HKDF-Extract(0, PSK或0)
handshake_secret = HKDF-Extract(early_secret, ECDHE共享密钥)
master_secret = HKDF-Extract(handshake_secret, 0)
各阶段流量密钥由对应secret派生。

0-RTT机制：服务器在NewSessionTicket中发送PSK和票据。客户端后续连接在ClientHello中携带PSK标识，并立即用early_traffic_secret加密应用数据（如HTTP请求）。该数据不具备前向保密且可被重放，故只能用于幂等请求。

与前端概念的对比：前端开发者熟悉的HTTP缓存与TLS 0-RTT都旨在减少往返，但HTTP缓存是资源复用，不涉及密码学状态；TLS 0-RTT是密钥状态复用，需重放防护。类似地，前端中的JWT是应用层凭证，而TLS PSK是传输层密钥材料，层次完全不同。

### 3. 基础代码与实战验证
```text
以下Python代码使用标准库模拟TLS 1.3密钥派生的核心逻辑（HKDF），不依赖任何框架：

import hashlib, hmac, os

def hkdf_extract(salt, ikm):
    # 提取伪随机密钥（PRK），salt为None时使用全零字节
    if salt is None:
        salt = bytes(32)
    return hmac.new(salt, ikm, hashlib.sha256).digest()

def hkdf_expand(prk, info, length):
    # 扩展PRK到所需长度的输出密钥材料
    t = b''
    okm = b''
    i = 1
    while len(okm) < length:
        t = hmac.new(prk, t + info + bytes([i]), hashlib.sha256).digest()
        okm += t
        i += 1
    return okm[:length]

# 模拟ECDHE产生的共享密钥（真实实现为椭圆曲线点乘结果）
shared_secret = os.urandom(32)

# 双方随机数
client_random = os.urandom(32)
server_random = os.urandom(32)

# 无PSK时，early_secret由全零输入派生
early_secret = hkdf_extract(None, bytes(32))
# 握手密钥：结合ECDHE共享密钥
handshake_secret = hkdf_extract(early_secret, shared_secret)
# 应用密钥：从握手密钥再派生
master_secret = hkdf_extract(handshake_secret, bytes(32))
application_key = hkdf_expand(master_secret, b'application key', 16)

# 0-RTT场景：使用PSK直接生成early_traffic_secret
psk = os.urandom(32)
early_secret_psk = hkdf_extract(None, psk)
early_traffic_key = hkdf_expand(early_secret_psk, b'early traffic', 16)

每行注释解释对应底层运作。实际握手还涉及协商哈希、曲线选择、证书验证等，但密钥分层结构即如上所示。
```

### 4. 常见误区与进阶思考
误区1：认为0-RTT数据与普通TLS数据具有同等安全性。实际上0-RTT数据没有前向保密，且可被重放，因此只能用于幂等操作，应用层必须实现防重放机制。
误区2：混淆TLS握手与TCP握手。TLS握手运行在TCP连接之上，两者独立。一个HTTPS请求的延迟包含TCP三次握手、TLS握手（可能1-RTT或0-RTT）和HTTP数据传输，不能将TLS握手延迟等同于TCP连接建立延迟。

思考题：在TLS 1.3完全握手中，若客户端与服务器协商的key_share曲线不匹配，服务器会返回HelloRetryRequest。该消息本身未加密，那么其完整性如何保证？请结合握手消息哈希链与CertificateVerify的覆盖范围，解释为什么无需单独签名。
