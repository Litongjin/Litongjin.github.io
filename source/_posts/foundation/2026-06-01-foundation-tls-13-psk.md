---
title: "每日基础技术总结 · 2026-06-01 · TLS 1.3 的会话恢复与 PSK 预共享密钥"
date: 2026-06-01 08:00:00
categories: [技术分享]
tags: ["技术分享", "安全基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-06-01 · TLS 1.3 的会话恢复与 PSK 预共享密钥

## 📚 今日主题

> **TLS 1.3 的会话恢复与 PSK 预共享密钥**（安全基础）

### 1. 核心概念速览
TLS 1.3 的会话恢复（Session Resumption）是一种利用预共享密钥（PSK，Pre-Shared Key）跳过完整握手流程的机制。其本质是：将上一次握手协商出的密钥材料（或外部预置的密钥）作为后续连接中认证和密钥派生的基础，通过服务器下发的 ticket 在客户端与服务器间传递状态，从而将握手延迟从一次完整的 (EC)DHE 往返降低为 0-RTT 或 1-RTT。它解决的核心问题是高延迟网络下短连接反复握手带来的性能损耗，以及服务端密钥协商的计算开销。机制上，服务器在完成握手后发送 NewSessionTicket，内含一个 PSK 标识（ticket）和对应的密钥；客户端缓存该信息，在后续连接中于 ClientHello 的 pre_shared_key 扩展中携带该标识和一个基于 PSK 计算的绑定值（binder），服务器验证后即可复用 PSK 派生会话密钥，不再进行公钥运算。在整个计算机体系中，TLS 1.3 位于 TCP 之上、应用层之下，是传输层安全协议的事实标准；会话恢复是 TLS 性能优化的重要支柱，直接决定连接建立的延迟、吞吐和服务端可扩展性。专业工程师必须掌握它，因为 HTTPS、gRPC、WebSocket 等现代应用都建立在其上，不理解 PSK 机制就无法设计低延迟系统、排查握手失败，也无法正确评估 0-RTT 的安全边界。

### 2. 底层原理剖析
TLS 1.3 完整握手：客户端发送 ClientHello（随机数、支持的密码套件、密钥共享），服务器回复 ServerHello（选择套件、随机数、密钥共享），双方经 (EC)DHE 计算共享秘密，再通过 HKDF 逐级派生主密钥、握手密钥和应用流量密钥。握手结束后，服务器生成一个随机 PSK（或从主密钥派生），用服务器专属密钥加密（或绑定）后作为 ticket 通过 NewSessionTicket 消息发给客户端。客户端保存 ticket 和 PSK。

下次连接时，客户端构造 ClientHello，在 pre_shared_key 扩展中放入 ticket 标识和 binder。binder 是使用 PSK 对当前 ClientHello（不含 binder 字段）的完整消息哈希计算的 HMAC，用于向服务器证明客户端确实持有该 PSK。服务器收到后，根据 ticket 解密或查找出 PSK，用同样的方式重算 binder 并比对；若一致，则基于 PSK、客户端随机数和服务器随机数，通过 HKDF-Extract/Expand 派生早期数据密钥（用于 0-RTT）和握手密钥，后续握手消息及应用数据均由此加密。整个过程中，公钥密码学只在首次完整握手时使用，恢复时 PSK 直接参与密钥派生，不涉及 (EC)DHE，因此延迟极低。

与前端已有概念的对比：前端常用的 JWT 和 Cookie 同样具备“客户端保存凭证，下次携带”的表象，但本质完全不同。JWT 是无状态身份令牌，服务器仅验证签名，不保存任何会话状态，且 JWT 不参与任何加密密钥的推导；TLS 1.3 的 PSK 则是真正的密钥材料，它必须被用于 HKDF 派生，其泄露直接导致会话密钥可被计算。这类似于 TypeScript 的 interface 与 Java 的 interface：名称和表面用途相似（都描述结构约束），但 TS interface 只在编译期存在，运行时无迹可寻，而 Java interface 是运行时类型系统的组成部分，有虚方法表和动态分派。同理，JWT 和 PSK 的相似性仅限于“携带凭证”这一层，而在密码学角色和协议栈位置上完全是两回事。

### 3. 基础代码与实战验证
```text
下面以伪代码展示 TLS 1.3 会话恢复的核心逻辑，重点在于 PSK 的保存、binder 生成和服务器验证。

# 第一次完整握手结束后，服务器发送 NewSessionTicket
# 服务器端：生成随机 PSK，并用服务器密钥加密作为 ticket
psk = random_bytes(32)
ticket = encrypt_with_server_secret(psk)
send_to_client(NewSessionTicket(ticket_id = sha256(ticket), ticket = ticket))

# 客户端：保存 ticket 与 psk（psk 可从 ticket 解密得到，或由本地派生）
session_cache[ticket_id] = { 'psk': psk, 'ticket': ticket }

# 第二次连接，客户端构造 ClientHello
client_hello = build_client_hello()  # 包含随机数、支持的组等
# 关键：计算 binder 前必须先确定不含 pre_shared_key 扩展的 ClientHello 完整哈希
binder_input = transcript_hash(client_hello_without_psk_extension)
binder = HMAC(psk, binder_input)
# 将 ticket_id 和 binder 放入 pre_shared_key 扩展
client_hello.extensions['pre_shared_key'] = { 'ticket_id': ticket_id, 'binder': binder }
# 若启用 0-RTT，此时可直接发送 Early Data（用早期密钥加密应用数据）

# 服务器验证
recv_ticket_id = client_hello.extensions['pre_shared_key']['ticket_id']
psk = decrypt_with_server_secret(recv_ticket_id)  # 从 ticket 解出 PSK
expected_binder = HMAC(psk, transcript_hash(client_hello_without_psk_extension))
if constant_time_compare(binder, expected_binder):
    # 验证通过，基于 PSK 和双方随机数派生握手密钥
    handshake_secret = HKDF_Expand(psk, transcript_hash(server_hello), 'derived')
    # 继续完成握手，后续所有握手消息用该密钥加密
else:
    abort(handshake_failure)

关键点：binder 是 PSK 所有权的密码学证明，它被绑定到当前 ClientHello 的完整消息哈希，任何对 ClientHello 的篡改都会导致 binder 不匹配；PSK 是 256 位随机数，必须严格保密，任何泄露都等同于会话劫持。
```

### 4. 常见误区与进阶思考
误区一：将 PSK 等同于 JWT 或 session ID。JWT 和 session ID 是身份凭证，服务器只需验证签名或比对存储；PSK 是密钥材料，它必须参与 HKDF 派生，且其安全性直接影响所有后续密钥。即便 PSK 不直接加密应用数据，泄露 PSK 也能让攻击者重放 0-RTT 数据或计算握手密钥。

误区二：忽视 0-RTT 的重放风险。0-RTT 数据使用 PSK 派生的早期密钥加密，但服务器在完成握手前无法确认该 ClientHello 是否被重放；攻击者可以截获并重放整个 ClientHello 和 Early Data，导致服务端重复处理请求。因此 0-RTT 只能用于幂等操作，或必须由应用层设计防重放机制（如单调序号、时间戳校验）。

思考题：如果服务器采用无状态 ticket（将 PSK 加密后直接发给客户端），那么服务器需要定期更换 ticket 加密密钥。假设你负责该密钥的轮换，如何设计策略使得旧 ticket 在过期后仍能在一个宽限窗口内被正确解密，而不至于因密钥轮换导致所有用户瞬间掉线？这需要你从 ticket 的结构、密钥版本标识和缓存淘汰机制三个层面回答，背后是对 TLS 1.3 会话恢复容错性的真实工程理解。
