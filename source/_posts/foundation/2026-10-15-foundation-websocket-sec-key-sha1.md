---
title: "每日基础技术总结 · 2026-10-15 · WebSocket升级请求中的Sec-WebSocket-Key生成与SHA-1哈希验证过程"
date: 2026-10-15 07:15:26
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-15 · WebSocket升级请求中的Sec-WebSocket-Key生成与SHA-1哈希验证过程

## 📚 今日主题

> **WebSocket升级请求中的Sec-WebSocket-Key生成与SHA-1哈希验证过程**（网络基础）

### 1. 核心概念速览
Sec-WebSocket-Key 是 WebSocket 协议升级（RFC 6455）中由客户端生成的 16 字节随机数的 Base64 编码字符串。其本质是一个 nonce，用于配合固定 GUID 通过 SHA-1 哈希生成 Sec-WebSocket-Accept，以证明服务端确实理解 WebSocket 协议，且响应并非由缓存代理或错误配置产生。它解决的核心问题是：在 HTTP Upgrade 机制下，确保握手请求与响应之间的关联性，防止跨协议攻击与缓存投毒。机制上，客户端生成不可预测的 16 字节随机数并 Base64 编码；服务端将该 Key 与固定 GUID '258EAFA5-E914-47DA-95CA-C5AB0DC85B11' 拼接后计算 SHA-1，再 Base64 编码得到 Accept 返回；客户端验证该 Accept。该机制位于网络协议层，是 WebSocket 脱离 HTTP 语义前的最后一步升级校验，属于无状态挑战/响应模式。在 AI 体系中，WebSocket 常用于流式推理结果推送与训练指标实时回传，因此理解其握手安全机制是构建可靠实时数据链路的必要基础。

### 2. 底层原理剖析
底层流程：
1. 客户端生成 16 字节加密安全随机数（RFC 6455 要求必须为不可预测），进行 Base64 编码，得到形如 'dGhlIHNhbXBsZSBub25jZQ==' 的 Sec-WebSocket-Key。
2. 客户端发送 HTTP GET 请求，包含 Upgrade: websocket, Connection: Upgrade, Sec-WebSocket-Version: 13, Sec-WebSocket-Key。
3. 服务端收到后，提取 Key，与固定 GUID 字符串 '258EAFA5-E914-47DA-95CA-C5AB0DC85B11' 进行字符串拼接（Key 在前，GUID 在后，无分隔符）。
4. 对拼接后的字符串计算 SHA-1 哈希（输出 20 字节）。
5. 对哈希结果进行 Base64 编码，得到 Sec-WebSocket-Accept，返回 101 Switching Protocols 响应。
6. 客户端收到响应后，用同样的算法重算 Accept 并与服务端返回的 Accept 比较，若不一致则终止握手。

伪代码：
function computeAccept(key):
    GUID = '258EAFA5-E914-47DA-95CA-C5AB0DC85B11'
    sha1Bytes = SHA1(key + GUID)
    return Base64Encode(sha1Bytes)

Base64 编码细节：16 字节随机数编码后长度为 24（含 '=' 填充），SHA-1 的 20 字节摘要编码后长度为 28（含一个 '='）。服务端必须校验 Sec-WebSocket-Key 的 Base64 解码后严格为 16 字节。

与前端已有概念的对比：它与 HTTP Digest 认证的 nonce 相似，都是协议方提供的随机数，但 Sec-WebSocket-Key 由客户端生成，且不携带时间窗或密钥信息；它与 CSRF Token 类似用于绑定请求上下文，但目的不是防 CSRF，而是防缓存代理与协议降级。与前端工程中常见的 API Key 或 JWT 完全不同：后者携带身份信息，而 Sec-WebSocket-Key 是纯随机值，不提供身份认证。

### 3. 基础代码与实战验证
```text
const crypto = require('crypto');

// 客户端：生成 16 字节加密安全随机数，并 Base64 编码
const key = crypto.randomBytes(16).toString('base64');
console.log('Sec-WebSocket-Key:', key);

// 服务端：固定 GUID 是 WebSocket 协议规定的握手哈希后缀
const WS_GUID = '258EAFA5-E914-47DA-95CA-C5AB0DC85B11';

// 服务端计算 Sec-WebSocket-Accept：Key + GUID 拼接后做 SHA-1，再 Base64
const accept = crypto.createHash('sha1').update(key + WS_GUID).digest('base64');
console.log('Sec-WebSocket-Accept:', accept);

// 客户端验证：重算并比较（生产环境应使用 crypto.timingSafeEqual 防时序侧信道）
const expected = crypto.createHash('sha1').update(key + WS_GUID).digest('base64');
console.log('客户端验证通过:', expected === accept);
```

### 4. 常见误区与进阶思考
常见误区：
1. 误以为 Sec-WebSocket-Key 是加密密钥或身份凭证；实际上它是明文传输的 nonce，不提供机密性、完整性与身份认证。其作用是强制服务器证明自己理解 WebSocket 协议，防止缓存代理返回缓存的 101 响应。
2. 误认为此处的 SHA-1 哈希存在安全弱点（碰撞攻击）因而影响 WebSocket 安全。实际上这里 SHA-1 仅作为基于固定 GUID 的哈希校验，不依赖抗碰撞性，碰撞攻击不会导致握手伪造或协议降级，因此在该场景下 SHA-1 的安全性足够。

进阶思考题：
既然固定 GUID 与 SHA-1 算法是公开的，且 Sec-WebSocket-Key 本身也是明文传输的，那么任何中间人都可以计算出正确的 Sec-WebSocket-Accept。请解释：该机制在无 TLS 的情况下到底能防御什么攻击，不能防御什么攻击？为什么真正的安全性必须依赖 TLS？
