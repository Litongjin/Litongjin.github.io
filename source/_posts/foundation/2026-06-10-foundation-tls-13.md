---
title: "每日基础技术总结 · 2026-06-10 · TLS 1.3 握手与密钥协商"
date: 2026-06-10 08:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-06-10 · TLS 1.3 握手与密钥协商

## 📚 今日主题

> **TLS 1.3 握手与密钥协商**（网络基础）

### 1. 核心概念速览
TLS 1.3 握手是客户端与服务器在传输层安全协议中协商会话密钥、认证对方身份并建立加密通道的完整过程，核心由 RFC 8446 定义。它解决的核心问题是：在不安全的信道上，如何通过非对称密码学实现前向安全的对称密钥协商，同时完成双向身份认证，并抵御降级攻击、重放攻击和中间人攻击。其本质是结合了临时 Diffie-Hellman（ECDHE）密钥交换、证书签名验证和 HKDF 密钥派生的一次性握手状态机。在计算机体系中，它位于应用层（HTTP/2、HTTP/3）之下、TCP（或 QUIC/UDP）之上，是互联网数据机密性与完整性的基石。专业工程师必须掌握它，因为任何后端服务、微服务通信、API 网关乃至 AI 推理的模型分发都依赖 TLS，理解其握手过程是诊断性能瓶颈、配置安全策略、设计零信任架构的前提。

### 2. 底层原理剖析
TLS 1.3 握手的底层机制可拆解为三阶段：协商、认证与派生。

1. 协商阶段：客户端发送 ClientHello，携带支持的密码套件列表、key_share（客户端临时 ECDHE 公钥）、随机数 nonce 及扩展（如 SNI）。服务器选择密码套件后回复 ServerHello，携带选定的密码套件、服务器临时 ECDHE 公钥和 nonce。此时双方已拥有彼此的临时公钥，可各自计算出共享秘密（ECDHE 结果）。注意：TLS 1.3 的密码套件不再包含认证算法，认证始终通过证书签名完成，且所有套件均要求前向安全。

2. 认证阶段：服务器发送 EncryptedExtensions（证书请求等）、Certificate（服务器证书链）、CertificateVerify（对握手上下文做数字签名），最后发送 Finished（对全部握手消息的 HMAC）。客户端验证证书链和签名后，也发送 Certificate 与 CertificateVerify（若需双向认证），并发送 Finished。之后双方即可发送应用数据。

3. 密钥派生：核心使用 HKDF-Extract 与 HKDF-Expand。初始共享秘密（ECDHE 输出）经 HKDF-Extract 生成中间密钥，再通过 HKDF-Expand 以不同标签（如 'c hs traffic'、's hs traffic'、'c ap traffic' 等）派生出各阶段的流量密钥。TLS 1.3 的『密钥调度』是严格单向的：早期秘密 -> 握手秘密 -> 主秘密 -> 应用数据秘密，每步使用不同的哈希标签，确保密钥隔离。Finished 消息使用对应方向握手密钥计算，用于确认双方拥有相同的视图。

关键机制对比：TLS 1.3 放弃了 RSA 密钥交换（因为不具备前向安全），只保留 ECDHE；同时将大部分握手消息加密（EncryptedExtensions 之后），只有 ClientHello 和 ServerHello 明文。这与前端工程中的接口对比：Java 的接口是编译期类型契约，强调实现结构的强制一致；TS 的接口是结构类型系统，强调鸭子类型（只要形状相同即可）。而 TLS 握手不是类型契约，而是状态机协议——它要求双方严格按顺序发送消息并验证每个消息的指纹（Finished），任何顺序错误或指纹不匹配都会中止握手。更贴近的类比是：它类似于前后端 API 的双向签名校验（如 HMAC），但密钥本身通过临时密钥交换动态生成，而非静态预共享。

### 3. 基础代码与实战验证
由于 TLS 1.3 握手由操作系统和加密库实现，真实代码无法直接展示底层字节流。以下使用 Node.js 的 TLS 模块作为最小化验证，同时给出核心过程的精确伪代码。

```javascript
// 使用 Node.js 内置 crypto 与 tls 模块验证 TLS 1.3 握手
const tls = require('tls');
const fs = require('fs');

// 服务器端：创建 TLS 1.3 服务器，强制使用 TLS 1.3
const options = {
  key: fs.readFileSync('server.key'),
  cert: fs.readFileSync('server.crt'),
  minVersion: 'TLSv1.3',
  maxVersion: 'TLSv1.3'
};

const server = tls.createServer(options, (socket) => {
  // 握手完成后触发 secureConnect（客户端侧）或连接回调（服务器侧）
  console.log('服务器获取协议版本:', socket.getProtocol()); // 应输出 TLSv1.3
  console.log('服务器协商的密码套件:', socket.getCipher()); // 如 TLS_AES_256_GCM_SHA384

  // 此时应用数据已加密，底层密钥已由握手完成协商
  socket.on('data', (data) => {
    // 收到的数据已经解密，此处为应用层明文
    console.log('服务器收到:', data.toString());
    socket.end('pong');
  });
});

server.listen(8443, () => {
  // 客户端：连接到服务器，触发握手
  const client = tls.connect({
    port: 8443,
    servername: 'localhost', // SNI 扩展，用于服务器选择证书
    rejectUnauthorized: false // 仅为演示，不校验证书链
  }, () => {
    console.log('客户端协议版本:', client.getProtocol());
    console.log('客户端密码套件:', client.getCipher());
    client.write('ping');
  });
});
```

底层握手过程的伪代码（精确描述每步操作）：

```
客户端生成随机数 r_c，临时密钥对 (sk_c, pk_c)
构造 ClientHello { version=1.3, random=r_c, key_share=pk_c, cipher_suites=[...] }
发送 ClientHello

服务器生成随机数 r_s，临时密钥对 (sk_s, pk_s)
收到 ClientHello 后，选择密码套件，计算共享秘密 ss = ECDHE(sk_s, pk_c)
构造 ServerHello { version=1.3, random=r_s, key_share=pk_s, cipher_suite=selected }
发送 ServerHello

服务器基于 ss 派生握手密钥 hs
使用 hs 加密以下消息：
  EncryptedExtensions { } // 可包含应用层协议协商 ALPN
  Certificate { server_cert }
  CertificateVerify { signature = Sign(sk_cert, transcript_hash) }
  Finished { mac = HMAC(hs, transcript_hash) }
发送全部加密消息

客户端收到后，用 ECDHE(sk_c, pk_s) 计算相同的 ss，派生 hs
解密消息，验证证书链，验证签名，验证 Finished

客户端发送自己的 Finished（若需要客户端证书，则先发 Certificate + CertificateVerify）
双方各自派生应用数据密钥 ap
开始加密通信
```

上述代码中，`getProtocol()` 和 `getCipher()` 返回的值直接来自底层握手的协商结果，验证了 TLS 1.3 确实被使用。若强制 TLS 1.3 但服务器或客户端不支持，则会抛出异常，说明协议版本协商失败。

### 4. 常见误区与进阶思考
1. 误区：认为 TLS 1.3 握手总是 1-RTT。实际 TLS 1.3 支持 0-RTT（PSK 预共享密钥）恢复机制，但 0-RTT 数据存在重放攻击风险，需要应用层额外处理幂等性。工程师常误以为所有 TLS 1.3 连接都是快速握手，忽略了 0-RTT 的安全约束，可能导致在非幂等接口（如支付创建订单）中错误启用 0-RTT，造成重复扣款等事故。

2. 误区：混淆会话恢复与密钥更新。TLS 1.3 有独立的 KeyUpdate 消息，可在连接过程中更新流量密钥，而不需要重新握手。但许多工程师认为密钥只能通过新握手更换，或者误将 KeyUpdate 当作会话恢复。实际上 KeyUpdate 是单向的，双方各自独立发送，且密钥更新后旧密钥立即失效，这是实现长连接安全的关键机制，但很少被正确配置或测试。

思考题：在 TLS 1.3 中，若攻击者截获了 ServerHello 并篡改其中的 key_share，但客户端无法立即检测（因为该消息尚未被签名或加密），那么客户端最终如何必然发现这次篡改？请从密钥派生和 Finished 消息的验证链路给出精确解释——篡改会导致共享秘密不同，进而导致双方派生的握手密钥不同，最终服务器的 Finished 无法被客户端用自身密钥验证，握手终止；请具体说明哪一步先失败，以及这是否为可证明安全的机制。
