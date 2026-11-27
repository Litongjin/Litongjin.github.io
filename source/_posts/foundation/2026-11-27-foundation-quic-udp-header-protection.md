---
title: "每日基础技术总结 · 2026-11-27 · HTTP/3 QUIC协议基于UDP的头保护与加密头部设计"
date: 2026-11-27 09:50:54
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-27 · HTTP/3 QUIC协议基于UDP的头保护与加密头部设计

## 📚 今日主题

> **HTTP/3 QUIC协议基于UDP的头保护与加密头部设计**（后端基础）

### 1. 核心概念速览
HTTP/3 是 HTTP 语义在 QUIC 上的映射；QUIC 是运行在 UDP 之上、内嵌 TLS 1.3 的传输协议。所谓『基于 UDP 的头保护与加密头部设计』的本质不是把整个 IP/UDP/QUIC 头全部加密，而是将包中必须明文路由的字段（版本、Connection ID、长度等）保留为可解析，同时用独立的 header protection 算法掩码包号及首字节低位，并用 AEAD 加密包载荷。它解决三类问题：TCP 中间盒僵化与内核迭代慢、TCP 队头阻塞、被动观察者通过包号/握手指纹做链路关联。它位于传输层与应用层交界，是理解 HTTP/3 性能、0-RTT、连接迁移、抓包排障和安全边界的基础。

### 2. 底层原理剖析
1. UDP 选择：QUIC 把 UDP 当作最小端点传输 substrate，不依赖内核 TCP 状态；可靠性、拥塞控制、流复用、连接迁移、加密握手均在用户态实现。UDP 四元组可被 Connection ID 抽象，路径变化时连接不重置。
2. 包头分层：Long Header 用于 Initial/Handshake/0-RTT/Retry/Version Negotiation，Short Header 用于 1-RTT。Version、DCID/SCID、Length 等明文字段用于端点解复用；包号不直接明文。Retry 与 Version Negotiation 有特殊规则，不走普通 AEAD/HP 流程。
3. 密钥体系：Initial 密钥由客户端首个 DCID 通过 HKDF-SHA256 派生，保证无服务器状态即可处理首包；后续 Handshake/Application 密钥来自 TLS 1.3。每个方向派生 quic key、quic iv、quic hp、quic ku，分别用于 AEAD、nonce、头保护、密钥更新。
4. AEAD：加密对象是 payload；AEAD 的 AAD 是恢复后的包头（含真实包号）。nonce = quic_iv XOR packet_number。包号必须被 AEAD 间接认证，否则可被篡改导致解密失败或重放窗口异常。
5. Header Protection：AEAD 之后，从密文区固定位置取 16 字节 sample，位置为包号字段起始后 4 字节；用 hp_key 计算 5 字节 mask（AES-ECB 或 ChaCha20）。first_byte 与 mask[0] 按包类型掩码：Long Header 保护低 4 位，Short Header 保护低 5 位；packet_number 字段逐字节与 mask[1..] 异或。接收方先用可明文字段定位 sample，去掩码得到包号长度和真实包号，再做 AEAD 解密。
6. 与前端已有体系对比：浏览器里 HTTP/2 常是 TCP + TLS，TCP/IP 头、TLS record 头部分可见，HTTP 帧在 TLS 记录载荷中加密；QUIC 则把加密推进到传输包头本身，但只保护必要字段。类似 TypeScript interface 与 Java interface：名字相近，但前者是编译期结构类型、运行期擦除，后者是运行期 nominal 契约；QUIC header protection 看似『加密头部』，实际不是 AEAD，也不是 HTTP 头加密，而是为了隐藏包号的独立掩码。

### 3. 基础代码与实战验证
```text
# 目标：演示 QUIC v1 发送端如何生成受保护包，以及接收端如何恢复包号。
# 依赖抽象：HKDF、AEAD（如 AES-128-GCM）、HP（AES-ECB 或 ChaCha20）。

# 1) Initial 密钥派生：客户端用目的连接标识作为输入密钥材料。
initial_secret = HKDF_Extract(salt=QUIC_V1_INITIAL_SALT, ikm=client_dcid)
client_secret = HKDF_Expand(initial_secret, label='client in', length=32)
key = HKDF_Expand(client_secret, label='quic key', length=16)  # AEAD 密钥
iv  = HKDF_Expand(client_secret, label='quic iv', length=12)   # 包号 nonce 基底
hp  = HKDF_Expand(client_secret, label='quic hp', length=16)   # 头保护密钥，独立于 AEAD 密钥

# 2) 发送端：先构造可被 AEAD 认证的包头，再隐藏包号。
def send_packet(header_prefix, pn, payload, key, iv, hp):
    pn_len = minimal_pn_length(pn)              # QUIC 包号编码为 1~4 字节
    pn_bytes = encode_pn(pn, pn_len)
    first_byte = header_prefix[0]               # 首字节包含包类型与未保护高位
    first_byte |= (pn_len - 1) & 0x03           # 低两位编码包号长度，稍后会被掩码
    header = bytes([first_byte]) + header_prefix[1:] + pn_bytes
    nonce = xor_12(iv, pn)                      # AEAD nonce：IV 与包号异或，保证每包唯一
    ciphertext = AEAD_Seal(key, nonce, header, payload)  # AAD 包含真实包头与真实包号
    sample = (pn_bytes + ciphertext)[4:20]      # RFC: sample 从包号字段后第 4 字节开始，共 16 字节；不足需填充。
    mask = HP(hp, sample)                       # AES-ECB 或 ChaCha20 生成掩码；至少取 5 字节。
    protected_first = first_byte ^ (mask[0] & 0x0F if is_long_header(first_byte) else mask[0] & 0x1F)  # 长头保护低 4 位，短头保护低 5 位。
    protected_pn = xor_bytes(pn_bytes, mask[1:1+pn_len])  # 包号字段被掩码，旁观者无法观察单调递增。
    return bytes([protected_first]) + header[1:-pn_len] + protected_pn + ciphertext

# 3) 接收端：先根据明文字段定位 sample，再反掩码，最后 AEAD 解密。
def receive_packet(packet, hp, key, iv):
    pn_offset = parse_unprotected_header(packet)  # 解析 Version/Connection ID/Length 等，不依赖包号。
    sample = packet[pn_offset + 4 : pn_offset + 20]  # 与发送端一致：包号起始后 4 字节取 16 字节。
    mask = HP(hp, sample)                       # 使用同一方向的 hp 密钥。
    first = packet[0] ^ (mask[0] & (0x0F if is_long_header(packet[0]) else 0x1F))  # 恢复首字节低位。
    pn_len = (first & 0x03) + 1                 # 低两位给出包号长度。
    protected_pn = packet[pn_offset : pn_offset + pn_len]
    pn_bytes = xor_bytes(protected_pn, mask[1:1+pn_len])  # 恢复真实包号。
    header = bytes([first]) + packet[1:pn_offset] + pn_bytes
    ciphertext = packet[pn_offset + pn_len:]
    nonce = xor_12(iv, decode_pn(pn_bytes))     # 用恢复出的包号构造 AEAD nonce。
    payload = AEAD_Open(key, nonce, header, ciphertext)  # AAD 包含恢复后的包头；认证失败即丢包。
    return header, payload
```

### 4. 常见误区与进阶思考
1. 把 QUIC 头保护等同为完整头部加密或 HTTP 头加密。实际 Version、DCID/SCID、Length 等通常明文，用于路由/解复用；HP 只保护包号及首字节低位；HTTP 请求头是在 QUIC 流载荷中由 TLS/AEAD 保护并经 QPACK 压缩。抓包看到 DCID 不等于泄露应用数据。
2. 认为头保护提供完整性或替代 AEAD。HP 只是对被动观察者的掩码，防止通过包号序列推断丢包、重传、拥塞状态；防篡改与防重放依赖 AEAD、包号空间、密钥阶段与 ACK 机制。若把 HP 当作安全边界，会误判攻击模型。
进阶思考：既然 AEAD 的 AAD 最终包含恢复后的真实包号，为什么 QUIC 不直接把包号放进加密 payload，而必须设计一个独立的头保护层？请从接收方解密前必须知道 nonce/AAD、路由器需要解复用、以及中间盒不可修改明文包号这几个约束推导。
