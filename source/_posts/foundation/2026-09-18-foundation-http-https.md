---
title: "每日基础技术总结 · 2026-09-18 · HTTP/HTTPS 握手与加密过程"
date: 2026-09-18 07:02:44
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-18 · HTTP/HTTPS 握手与加密过程

## 📚 今日主题

> **HTTP/HTTPS 握手与加密过程**（前端底层与计算机基础）

### 1. 核心概念速览
HTTP 是应用层无状态请求/响应协议，默认基于 TCP，端口 80，报文明文。HTTPS 不是独立协议，而是 HTTP over TLS/SSL，默认端口 443，在 TCP 之上、HTTP 之下插入 TLS 记录层与握手层，提供机密性、完整性、身份认证。

核心问题：明文 HTTP 可被链路上任意中间人窃听、篡改、冒充。机制：通过 X.509 证书链和 CA 签名验证服务端身份；通过非对称密钥交换（ECDHE）协商出只有双方知道的共享秘密；通过 HKDF/PRF 派生出对称会话密钥；后续 HTTP 报文用 AEAD 对称加密（AES-GCM/ChaCha20-Poly1305）传输，并用 MAC 保证完整性。

位置：TCP/IP 协议栈中位于传输层与应用层之间的安全子层，是浏览器、API 网关、微服务、AI 模型下载与推理 API 的默认安全边界。前端工程师必须掌握：浏览器网络面板、证书错误、CORS、HTTP/2/3、CDN、Service Worker、WebSocket 安全、性能优化（TLS 握手 RTT）都直接受其影响。

### 2. 底层原理剖析
TLS 握手发生在 TCP 三次握手之后，目标是在不可信信道上建立可信的加密会话。

TCP 三次握手：
C -> S: SYN(seq=x)
S -> C: SYN+ACK(seq=y, ack=x+1)
C -> S: ACK(ack=y+1)

TLS 1.3 1-RTT 握手：
1. C -> S: ClientHello(random_C, cipher_suites, SNI, key_share, supported_versions, ALPN)
2. S -> C: ServerHello(random_S, cipher_suite, key_share)
3. S -> C: {EncryptedExtensions}, {Certificate}, {CertificateVerify}, {Finished}
4. C: 验证证书链：用系统根 CA 公钥逐级验签，检查域名、有效期、吊销；用自己临时私钥和服务器 key_share 计算 ECDHE 共享秘密。
5. C: 用 HKDF-Extract/Expand 从共享秘密和 transcript hash 派生 client_handshake_traffic_secret、server_handshake_traffic_secret、application_traffic_secret。
6. C -> S: {Finished}，包含握手消息哈希的 HMAC，验证握手完整性。
7. C <-> S: 用 application_traffic_secret 派生对称密钥，AEAD 加密 HTTP 请求/响应。

TLS 1.2 差异：ServerHello 后服务器发送 Certificate、ServerKeyExchange（ECDHE 参数）、ServerHelloDone；客户端发送 ClientKeyExchange、ChangeCipherSpec、Finished；服务器发送 ChangeCipherSpec、Finished；密钥导出用 PRF。证书在 TLS 1.2 中明文传输，TLS 1.3 中证书在 ServerHello 后立即被加密。

对称加密：AES-GCM/ChaCha20-Poly1305 提供机密性+完整性，nonce 由密钥和序列号构成，防重放。前向保密：ECDHE 临时密钥，长期私钥仅用于签名证书验证，不用于密钥交换。证书验证链：叶子证书 -> 中间 CA -> 根 CA，根 CA 预置在操作系统/浏览器信任库。签名算法如 RSA-PSS/ECDSA。

与前端已有概念对比：
- 与 JWT 验签同源：JWT 用非对称签名保证 payload 完整性和签发者身份；TLS 证书用 CA 签名保证公钥归属。区别：JWT 是应用层、无状态令牌；TLS 是传输层、有状态会话。
- 与 CORS 预检：都是「先协商后传输」，但 CORS 是浏览器同源策略的应用层许可，不提供加密；TLS 是传输层密码学协商，提供机密性、完整性、身份认证。
- 与 WebSocket 握手：WebSocket 通过 HTTP Upgrade 从 HTTP 切换到 WebSocket 协议；TLS 通过握手从明文 TCP 切换到加密信道。
- 与 TypeScript 接口：TS 接口是编译时静态类型约束，无运行时强制；TLS 证书是运行时密码学身份约束，有数学强制。
- 与 HTTP 缓存：HTTP 缓存是应用层语义，TLS 会话恢复是传输层优化。

### 3. 基础代码与实战验证
```text
// 使用 Node.js 内置 tls 模块验证 TLS 握手与证书链，无需第三方框架
// 运行：node tls-check.js
const tls = require('tls');
const CRLF = String.fromCharCode(13, 10); // HTTP 报文行分隔符 CRLF，TLS 加密在 TCP 之上
const socket = tls.connect({
  host: 'example.com',
  port: 443,
  servername: 'example.com', // SNI 扩展：明文告知服务器要访问的域名，服务器据此选择证书
  rejectUnauthorized: true   // 强制验证证书链，失败触发 error 事件，防止中间人
}, () => {
  console.log('authorized:', socket.authorized); // true 表示证书链已由系统根 CA 验签通过
  console.log('protocol:', socket.getProtocol()); // 协商结果，如 TLSv1.3
  console.log('cipher:', socket.getCipher());     // 协商的 AEAD 套件，如 TLS_AES_128_GCM_SHA256
  const cert = socket.getPeerCertificate(true);  // 获取叶子证书及完整链
  console.log('subject:', cert.subject);          // 证书持有者，验证域名匹配
  console.log('issuer:', cert.issuer);            // 签发者，逐级向上到根 CA
  // 握手完成后，写入的数据由 TLS 记录层用对称会话密钥加密，经 TCP 发送
  socket.write(['GET / HTTP/1.1', 'Host: example.com', 'Connection: close', '', ''].join(CRLF));
});
socket.on('data', (chunk) => {
  // 收到的是 TLS 解密后的 HTTP 响应明文；链路中间人只能看到密文
  console.log(chunk.toString().slice(0, 200));
});
socket.on('error', (err) => {
  console.error('TLS error:', err.message); // 证书过期/域名不匹配/自签名会在此暴露
});
socket.on('end', () => process.exit(0));

// 也可用 openssl s_client -connect example.com:443 -tls1_3 -showcerts 观察握手与证书链
```

### 4. 常见误区与进阶思考
常见误区：
1. 认为 HTTPS 加密了整个 HTTP 请求，包括 DNS、SNI、目标 IP。实际上：DNS 查询通常明文（除非 DoH/DoT），SNI 在 TLS 1.2/1.3 中默认明文（除非 ECH），目标 IP 和端口明文，TCP 握手明文。TLS 只加密 HTTP 应用数据，中间人仍可看到连接元数据。
2. 认为非对称加密用于加密所有数据。实际上非对称加密仅用于身份验证和密钥交换，数据用对称加密，因为非对称加密性能差、有长度限制。同时，认为有了 HTTPS 就绝对安全，忽略证书验证绕过（如 rejectUnauthorized: false）、中间人代理、证书吊销、弱密码套件、0-RTT 重放。

思考题：在 TLS 1.3 0-RTT 中，客户端用 PSK 恢复会话并直接发送 early data，为什么服务器必须实现 anti-replay 机制？如果 HTTP 请求是幂等的 GET，风险是否可接受？若换成非幂等的 POST（如支付），TLS 层和 HTTP 层分别应如何缓解？
