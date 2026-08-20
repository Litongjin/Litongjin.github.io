---
title: "每日基础技术总结 · 2026-08-10 · TLS 1.3 握手流程与 0-RTT：会话恢复与重放攻击风险"
date: 2026-08-10 08:00:00
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-08-10 · TLS 1.3 握手流程与 0-RTT：会话恢复与重放攻击风险

## 📚 今日主题

> **TLS 1.3 握手流程与 0-RTT：会话恢复与重放攻击风险**（后端基础）

### 1. 核心概念速览
TLS 1.3 是传输层安全协议的最新标准版（RFC 8446），其握手流程旨在以最少往返次数（1-RTT）完成密钥协商与身份认证，并支持会话恢复机制实现 0-RTT 数据传输。0-RTT 允许客户端在第一次握手中携带应用数据，本质是利用先前会话的 PSK（预共享密钥）加密数据，避免等待服务器响应。它解决的核心问题是降低连接建立延迟，尤其对 HTTP/3 和移动网络中的短连接至关重要。机制上，0-RTT 依赖会话票据（Session Ticket）或外部 PSK，服务器通过早数据（Early Data）扩展名识别并解密。该机制在网络安全体系中位于传输层与应用层之间，是后端服务高并发、低延迟架构的关键基础。专业工程师必须掌握它，因为 0-RTT 的语义与安全边界（重放防护、前向保密）直接影响 API 设计、幂等性策略和反重放中间件实现。

### 2. 底层原理剖析
TLS 1.3 握手核心流程分两个阶段：密钥交换与认证。

1. 首次完整握手（1-RTT）：
   - ClientHello：客户端发送支持的密码套件、随机数、以及密钥共享（Key Share，如 X25519 公钥）。
   - ServerHello：服务器选择密码套件，返回自己的密钥共享和随机数，随后计算握手密钥，发送 EncryptedExtensions、Certificate、CertificateVerify（对握手哈希的签名）、Finished。
   - 客户端验证证书链与签名，计算相同密钥，发送 Finished。双方开始应用数据加密。

2. 会话恢复（PSK）：
   - 服务器在首次握手后发送 NewSessionTicket，内含票据（Ticket）和随机生成的 PSK。客户端存储该票据。
   - 后续连接：客户端在 ClientHello 中携带 `pre_shared_key` 扩展，并附上票据标识和 `obfuscated_ticket_age`。服务器验证票据有效后，直接派生应用密钥，并立即发送 Finished（不需要证书验证）。此过程仅需 0-RTT（若客户端同时发 Early Data）。

3. 0-RTT 数据发送：
   - 客户端在 ClientHello 的 `early_data` 扩展中声明要发送早数据，并使用从 PSK 派生的 `client_early_traffic_secret` 加密应用数据。
   - 服务器收到后，若接受早数据，则解密并处理，然后正常完成握手；若拒绝，则返回 `EncryptedExtensions` 中不含 `early_data` 扩展，客户端需重发数据。

4. 重放风险本质：
   - 0-RTT 数据不经过服务器随机数（ServerHello）绑定，攻击者可截获 ClientHello 与 Early Data 并重放给多个服务器或同一服务器多次。
   - 因为 PSK 是静态的，重放的数据在服务器端会被解密并可能导致重复操作（如重复下单、重复转账）。
   - 防护机制：服务器必须实现重放窗口（记录 ClientHello 随机数或时间戳）、单次使用票据、或要求应用层幂等。

与前端知识的对比：
- 类似前端 HTTP 缓存协商（ETag/If-None-Match）与请求重放的关系：0-RTT 如同带预授权的缓存令牌，但缺乏服务端状态校验。
- 类似 Redux 中持久化 state 的恢复：PSK 是加密的 session snapshot，但恢复后没有强制验证 state 的有效期。
- 本质差异：TLS 是密码学协议，重放防护必须由协议层或应用层显式实现，而非像浏览器同源策略那样默认阻止。

### 3. 基础代码与实战验证
纯伪代码/步骤描述（无语言依赖，验证原理）：

```
// 客户端首次连接
client -> server: ClientHello { key_share: ephemeral_pub, supported_groups }
server -> client: ServerHello { key_share: server_pub, ciphersuite }
server -> client: EncryptedExtensions, Certificate, CertificateVerify, Finished
client -> server: Finished
// 服务器在 Finished 后发送会话票据
server -> client: NewSessionTicket { ticket, ticket_lifetime, obfuscated_ticket_age }

// 第二次连接，0-RTT 尝试
client -> server: ClientHello { pre_shared_key: ticket, early_data: true, key_share: new_ephemeral_pub }
client -> server: EarlyData { encrypted_application_data }  // 使用 client_early_traffic_secret 加密
server -> client: ServerHello { pre_shared_key: accepted }
server -> client: EncryptedExtensions (无 early_data 扩展，若拒绝则早数据不生效)
server -> client: Finished
client -> server: Finished
// 后续应用数据使用握手密钥

// 重放攻击演示
attacker: 截获上述 ClientHello + EarlyData
attacker -> server (victim): 重放相同的 ClientHello + EarlyData
server: 验证 ticket 仍有效，解密早数据，执行业务逻辑（若应用未做幂等） -> 重复扣款
```

关键注释：
- `obfuscated_ticket_age` 用于服务器判断票据是否新鲜，但攻击者可以精确重放原样数据。
- 服务器是否接受 0-RTT 取决于 `max_early_data_size` 和重放窗口，但协议本身不保证数据不被重放。
- 真实代码中，OpenSSL 提供 `SSL_CTX_set_early_data_enabled` 与 `SSL_read_early_data`，但应用层必须自行实现防重放逻辑。

### 4. 常见误区与进阶思考
误区一：认为 0-RTT 是安全的，因为 TLS 加密了。
事实：0-RTT 加密但不可防重放。加密只保证机密性，不保证唯一性。攻击者不需要解密，只需要原样转发数据包，服务器就会解密并执行。防护必须落在应用层（幂等键、时间戳校验）或使用服务器端单次票据（ticket 只允许使用一次，但会失去 0-RTT 的持久性）。

误区二：认为 TLS 1.3 的 0-RTT 与 TCP Fast Open 类似，可自然保证顺序和防重放。
事实：TFO 只优化 TCP 层连接建立，数据仍受 TCP 序列号与重传机制保护；而 0-RTT 数据在 TLS 层，没有 TLS 层重放保护。TCP 层会处理重复包，但不会处理来自不同连接（或多次连接）的相同应用数据。

思考题：
假设你的后端服务使用 TLS 1.3 0-RTT 接收客户端上报的埋点数据（无业务副作用），你是否还需要防重放？如果客户端发送的是『创建订单』请求，且你只在服务器内存中维护一个最近 5 秒内的请求哈希集合来防重放，请分析该方案的缺陷，并说明如何利用 TLS 层已有的 `obfuscated_ticket_age` 信息与客户端随机数构造更健壮的防重放机制。
