---
title: "每日基础技术总结 · 2026-08-30 · HTTP/HTTPS 握手与加密过程"
date: 2026-08-30 06:57:23
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-30 · HTTP/HTTPS 握手与加密过程

## 📚 今日主题

> **HTTP/HTTPS 握手与加密过程**（前端底层与计算机基础）

### 1. 核心概念速览
HTTP 是运行在 TCP 之上的无状态请求/响应应用层协议，本质上不具备安全属性。HTTPS 的本质是 HTTP over TLS，即由 TLS 在应用层与传输层之间提供四类安全服务：对端身份认证、机密性、完整性和重放防护。HTTPS 握手就是 TLS 握手协议的一次完整执行，目标是在通信双方不预先共享密钥的前提下，通过公钥密码学完成（至少服务端）身份认证，协商出攻击者无法从线路上推导出的对称会话密钥，并建立加密与完整性保护的数据通道。在体系结构上，TLS 位于 HTTP 之下、TCP 之上：HTTP 层看到的是可靠的字节流；TCP 层看到的是普通的 TLS 记录数据。专业工程师必须掌握它，因为安全边界并不由应用代码或框架决定，而由握手期间协商的密码套件、证书信任链和密钥派生方式决定；不了解握手过程就无法定位中间人攻击、证书配置错误、握手延迟、会话恢复失效等线上问题。

### 2. 底层原理剖析
一、两阶段连接
HTTPS 连接建立分两层完成：先做 TCP 三次握手（SYN/SYN-ACK/ACK），再在同一 TCP 连接上做 TLS 握手。没有 TLS 会话恢复时，新建连接需要多一次或两次网络往返（RTT）。

二、TLS 1.2 经典握手流程
1. ClientHello：客户端生成随机数 C，携带支持的 TLS 版本、密码套件列表和扩展（如 SNI、supported_groups、signature_algorithms）。这一步决定服务器如何选择证书和密钥交换算法。
2. ServerHello：服务器选定 TLS 版本和密码套件，生成随机数 S，并发送服务器证书；若使用 ECDHE，还会发送服务端临时公钥和签名。
3. 客户端验证证书链：从服务端证书追溯到本地信任根，校验有效期、吊销状态，以及主机名是否匹配证书中的 SAN。然后验证服务器对握手消息的签名。
4. 客户端发送 ClientKeyExchange：若使用 RSA 密钥交换，生成 48 字节 pre-master secret，用服务器公钥加密后发送；若使用 ECDHE，发送客户端临时公钥，双方各自通过 ECDH 算法算出相同的 pre-master secret。
5. 两端用 PRF(pre-master secret || C || S) 派生 master secret，再经 key expansion 派生出用于对称加密和 MAC 的会话密钥。
6. 客户端发出 ChangeCipherSpec（旧版本标志切换为加密模式）和 Finished（所有握手消息的哈希并用会话密钥加密）；服务器同样发送 ChangeCipherSpec 和 Finished。Finished 双方都能验证，确认握手消息没有被篡改且密钥可用。

三、TLS 1.3 的本质变化
TLS 1.3 握手压缩为 1-RTT：ClientHello 直接携带 key_share（客户端临时公钥），ServerHello 返回选中的 key_share；双方在本轮结束即可派生会话密钥，随后服务器发送的 Certificate 和 CertificateVerify 已经用会话密钥加密。它用 HKDF 替代 PRF，移除了 RSA 密钥交换和 CBC 模式，强制使用 ECDHE/PSK，因此前向保密成为必选。会话恢复使用 PSK，支持 0-RTT，但 0-RTT 需要应用层自行防重放。

四、与前端已有概念的对照
HTTPS 握手的“先协商安全参数、再传输业务数据”，在机制上与 CORS 预检请求（OPTIONS 预检）相似：先把前置条件达成，再发送实际请求。但 CORS 预检是浏览器同源策略下的策略协商，TLS 握手是密码学层的密钥协商和身份认证，二者层级与强度完全不同。更本质的对照是：TS 的 interface 在编译后消失，只存在于静态类型层；Java 的 interface 是运行时的多态契约。类似地，HTTP 语义在 HTTP 和 HTTPS 中完全一致，HTTPS 多出的安全属性来自外层 TLS 协议而非 HTTP 本身；只看 URL 前缀会忽略连接层真正决定安全的握手、证书和套件。

### 3. 基础代码与实战验证
```text
用 Python 标准库验证真实 TLS 握手后的加密通道。

import socket, ssl

ctx = ssl.create_default_context()
# 装载系统 CA 证书库；wrap_socket 握手时用这些根证书验证服务器证书链。
ctx.check_hostname = True
# 强制校验主机名与证书 SAN 匹配，防止中间人使用合法证书但域名不匹配。

raw_sock = socket.create_connection(("example.com", 443))
# 这一步只完成 TCP 三次握手；socket 上还没任何 TLS 记录。

with ctx.wrap_socket(raw_sock, server_hostname="example.com") as s:
    # wrap_socket 内部执行完整 TLS 握手：
    # ClientHello -> ServerHello -> Certificate -> ServerKeyExchange -> ClientKeyExchange -> Finished。
    # 回调返回时，session keys 已安装，socket 读取写入已经走 TLS 记录层的加解密。
    print("TLS 版本:", s.version())
    # 例如 TLSv1.3
    print("协商套件:", s.cipher()[0])
    # 例如 TLS_AES_256_GCM_SHA384

    s.send(b"GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n")
    # 应用层发送 HTTP 明文；数据经过 TLS 层时会被加密成 TLS 密文记录。

    response = s.recv(4096)
    # 内核返回的是 TLS 已经解密后的 HTTP 明文。
    print(response[:128].decode("utf-8", "ignore"))

这段代码不依赖任何 Web 框架，直接用 socket 操纵到传输层，再通过 SSLContext 触发一次真实的 TLS 握手，可验证 HTTPS 的本质是“TLS 提供加密字节流，HTTP 在该字节流上运行”。
```

### 4. 常见误区与进阶思考
误区一：把“HTTPS 握手”等同于“证书验证”。证书验证只是握手的一部分，且主要解决“服务器是否具有用户声称的身份”。即使证书链合法，如果密码套件是 RSA 密钥交换且服务端没有使用 ECDHE，攻击者拿到服务端私钥后仍能解密历史流量，因为私钥直接参与了 pre-master secret 的传输。也就是说，认证正确并不自动意味着会话保密，更不意味着前向保密。

误区二：认为握手完成后业务数据也全部使用公钥加密。真实机制是：公钥（或 ECDHE 临时公钥）只用于握手阶段的密钥协商和身份签名；业务数据统一使用握手后派生的对称密钥（如 AES-GCM）进行加密和完整性保护，因为对称加密在性能上远优于非对称加密。这个误区会让人误判“HTTPS 每次请求都做非对称加密”从而解释错误性能瓶颈。

进阶思考：假设攻击者获得了服务器的 RSA 私钥，但服务器使用的密码套件是 ECDHE-RSA-AES128-GCM-SHA256。攻击者能否解密之前被动记录的流量？请从 RSA 私钥在握手中的作用（签名证书/签名 ServerKeyExchange）和 ECDHE 密钥交换彼此独立这一点，分析前向保密为什么仍有效。
