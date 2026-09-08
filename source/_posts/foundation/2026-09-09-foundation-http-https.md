---
title: "每日基础技术总结 · 2026-09-09 · HTTP/HTTPS 握手与加密过程"
date: 2026-09-09 07:02:15
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-09 · HTTP/HTTPS 握手与加密过程

## 📚 今日主题

> **HTTP/HTTPS 握手与加密过程**（前端底层与计算机基础）

### 1. 核心概念速览
HTTP/HTTPS 握手是客户端与服务器在传输层之上建立会话、协商加密参数并验证身份的过程。HTTP 本身是无状态的应用层协议，其握手仅指 TCP 三次握手，用于建立可靠字节流连接。HTTPS 则是在 HTTP 与 TCP 之间插入 TLS 协议层，握手过程包含 TCP 三次握手和 TLS 握手，后者负责协商加密套件、交换密钥材料、验证服务器身份（可选验证客户端），最终生成会话密钥用于对称加密数据传输。其本质是解决明文信道上的三个问题：机密性、完整性、身份认证。TLS 握手设计的关键在于通过非对称加密和密钥协商算法（如 ECDHE/RSA）在不安全信道上安全地生成共享密钥，再用高吞吐的对称加密保护实际数据。该机制位于 OSI 第 4 层之上、应用层之下，是现代 Web 安全的基石。专业工程师必须掌握它，因为它是前端向后端传递数据的第一道安全边界，也是性能优化（TLS 握手延迟、会话复用）和问题排查（证书链、协议版本、加密套件）的核心知识域，更是理解 HTTP/2、HTTP/3、零信任架构的底层基础。

### 2. 底层原理剖析
HTTP 握手：客户端发送 SYN，服务器回应 SYN-ACK，客户端再发 ACK，完成 TCP 连接。之后可发送 HTTP 请求。HTTPS 握手在 TCP 连接之上叠加 TLS 握手，以 TLS 1.3 为例，本质步骤：1. ClientHello（客户端生成随机数 client_random，支持的加密套件列表，以及支持的 TLS 版本）；2. ServerHello（服务器选择加密套件，生成随机数 server_random）；3. 服务器发送其证书链（Certificate）；4. 服务器发送密钥交换参数（如 ECDHE 公钥），并签名证明拥有对应私钥（CertificateVerify）；5. 服务器发送 Finished；6. 客户端验证证书链与签名，计算预主密钥（通过 ECDHE 交换得到共享密钥），进而用 PRF（伪随机函数）结合 client_random 和 server_random 派生出会话密钥（主密钥）；7. 客户端发送 Finished，之后双方用对称密钥加密应用数据。TLS 1.3 将握手压缩为 1-RTT（往返时延），且移除了 RSA 密钥交换以支持前向保密。对前端工程师熟悉的概念，TLS 握手类似 Java 接口与 TypeScript 接口的本质区别：Java 接口是编译期强约束的契约，运行时存在方法签名与实现绑定，且可被反射访问；TypeScript 的接口是纯类型层面的结构约束，在编译后完全消失，无运行时存在。TLS 握手则像一种运行时协议契约：握手过程中的证书、随机数、密钥交换参数都是显式的消息，必须按顺序交换，且每一步都有严格的校验逻辑；而 HTTP 消息是无状态的，可独立处理。两者的共同点是都定义了交互双方必须遵守的格式和行为规则，但 TLS 的状态机是有会话状态的、分阶段的，前端 TS 接口则是静态的、无流程的。理解 TLS 握手本质要求区分两层：握手协议（用于协商）与记录协议（用于加密传输）。握手完成后，记录协议将数据分割成明文片段，加 MAC/填充，再用对称密钥（如 AES-GCM）加密并附加序列号以防止重放攻击。整个机制的底层依赖公钥基础设施（PKI），证书链验证信任根，这是与前端本地存储加密完全不同层次的安全模型。

### 3. 基础代码与实战验证
验证 HTTPS 握手的极简方式：使用 OpenSSL 和 curl 观察握手细节。

```bash
# 1. 观察 TLS 握手的完整消息序列（不发送应用数据，仅握手）
openssl s_client -connect example.com:443 -tls1_3 -msg -brief

# 2. 查看服务器证书链和选中的加密套件
echo | openssl s_client -connect example.com:443 2>/dev/null | grep -E 'Protocol|Cipher|Verify return'

# 3. 使用 curl 获取响应头，验证 HTTPS 工作正常，并显示 TLS 版本
curl -v --tlsv1.3 https://example.com 2>&1 | grep -E 'SSL|TLS'

# 4. 抓包观察 TCP 三次握手和 TLS 握手（需要 root）
# tcpdump -i any -w /tmp/tls.pcap port 443
```

关键点逐行注释：
- `-tls1_3` 强制使用 TLS 1.3，能观察到更简洁的握手消息（ClientHello, ServerHello, EncryptedExtensions, CertificateVerify, Finished）。
- `-msg` 打印每条握手消息的原始结构，验证真实传输顺序。
- `-brief` 输出精简的握手状态。
- `-cipher` 查看服务器从客户端提供的套件中选择的加密套件。

若无法运行网络命令，可用伪代码描述 TLS 1.3 握手状态机：

```
client -> server: {version: 1.3, client_random, key_share: g^x}
server -> client: {server_random, key_share: g^y, certificate: {identity, pubkey}}
server -> client: {certificate_verify: sign(pubkey, transcript)}
server -> client: {finished: encrypt(derive_secret(client_random, server_random, g^xy))}
client -> server: {finished: encrypt(derive_secret(...))}
双方使用相同的主密钥派生出 traffic_secrets，开始对称加密应用数据。
```

这段伪代码与真实抓包结果一一对应，能确认证书验证（CertificateVerify）发生在密钥交换参数之后、Finished 之前的顺序，以及前向保密由 ECDHE 的临时密钥对保证。

### 4. 常见误区与进阶思考
误区一：认为 HTTPS 握手只是 TCP 三次握手加一次非对称加密。实际上 TLS 握手涉及两轮往返（TLS 1.2 及以前）或一轮往返（TLS 1.3），且加密过程不只是非对称加密证书，关键是密钥协商（如 ECDHE）和 PRF 派生，非对称仅用于签名和身份认证（RSA 或 ECDSA），数据加密始终是对称加密。混淆这二者会导致难以理解证书的作用和性能瓶颈。

误区二：认为证书验证是验证域名与 IP 的绑定。实际上证书验证是验证证书链到根证书的信任锚，并检查域名是否包含在证书的 SAN（Subject Alternative Name）中，同时检查有效期和撤销状态。前端工程师容易将 HTTPS 误解为『加密后内容就安全』，但忽略身份认证，导致中间人攻击的风险认知缺失。

思考题：在 TLS 1.3 中，如果客户端和服务器在握手前已经共享一个静态密钥（例如通过带外方式预置），是否还需要 ClientHello/ServerHello 中的随机数和 ECDHE 交换？若去掉随机数，能否保证前向保密和会话唯一性？请从握手的抗重放、会话密钥派生的熵来源、以及前向保密三个角度分析，并说明在什么场景下这种设计会被接受。
