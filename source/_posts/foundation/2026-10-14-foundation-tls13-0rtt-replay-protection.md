---
title: "每日基础技术总结 · 2026-10-14 · HTTPS TLS 1.3 0-RTT模式的安全隐患与重放攻击防护机制"
date: 2026-10-14 07:15:57
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-14 · HTTPS TLS 1.3 0-RTT模式的安全隐患与重放攻击防护机制

## 📚 今日主题

> **HTTPS TLS 1.3 0-RTT模式的安全隐患与重放攻击防护机制**（网络基础）

### 1. 核心概念速览
TLS 1.3 0-RTT（Zero Round-Trip Time）是会话恢复模式：客户端基于先前握手获得的 PSK（Pre-Shared Key）在首个往返内加密并发送应用数据（early data），省去一个 RTT。其本质是把密钥协商从临时密钥交换提前到静态 PSK 派生：客户端在收到服务器随机数之前，就用仅由 PSK 和 ClientHello 派生出的 early traffic secret 加密数据。它解决低延迟重复连接问题，常用于 HTTPS 长连接优化、HTTP/3 等场景。专业工程师必须掌握：因为它在传输层引入了不可忽略的安全语义变化——0-RTT 数据不具备前向保密且天然可重放，若不理解其边界，会在 API 设计时错误暴露非幂等操作。

### 2. 底层原理剖析
底层机制：
1. 会话恢复：完整握手后服务器通过 NewSessionTicket 下发 PSK（含 ticket age add、ticket nonce 等）。
2. 0-RTT 发送：客户端在后续连接选择 PSK，计算 early_secret = HKDF-Extract(0, PSK)，再由 ClientHello 的 transcript hash 派生 client_early_traffic_secret，用其加密 early data，并紧接着发送 EndOfEarlyData。
3. 关键点：ServerHello.random 未参与派生，因此 0-RTT 密文不绑定当前连接的新鲜性；同一个 ClientHello 和 early data 可被原样重放。
4. TLS 本身不提供重放防护：RFC 8446 明确将重放防护责任交给应用层或服务器策略，例如仅允许幂等请求、单次使用票据、限制 ticket age 窗口、外部存储去重等。
与前端已有知识对比：类似 HTTP 条件请求（ETag/If-None-Match）中缓存验证可安全重放，因为请求本身不改变资源状态；而 0-RTT 重放等价于浏览器层透明重试某个 HTTP 请求，但缺少服务器提供的新鲜性挑战，因此比 HTTP/2 传输层重试更危险——HTTP/2 重试通常只对幂等方法自动执行，0-RTT 却可能被攻击者选择任意已捕获请求重放。

### 3. 基础代码与实战验证
```text
# --- 客户端：0-RTT 发送 ---
function client_0rtt_send(tickets, early_http_request):
    psk = select_psk(tickets)
    ch = build_client_hello()
    ch.extensions.append(pre_shared_key(psk.identity, obfuscated_age = psk.age + ticket_age_add))
    ch_hash = transcript_hash(ch)                 # 只含 ClientHello，不含服务器随机数
    early_secret = HKDF_Extract(0, psk.key)       # 0-RTT 密钥不来自 (EC)DHE
    c_early_ts = Derive_Secret(early_secret, 'c e traffic', ch_hash)
    key, iv = derive_aead_key_iv(c_early_ts)
    ciphertext = AEAD_Encrypt(key, iv, seq=0, plaintext=early_http_request)
    send(ch)
    send(ciphertext)
    send(EndOfEarlyData)

# --- 服务器：接收 early data 并检测重放 ---
function server_0rtt_handle(ch, ciphertext, psk_store):
    psk_id = ch.extensions.pre_shared_key.identity
    psk = psk_store[psk_id]
    ch_hash = transcript_hash(ch)
    early_secret = HKDF_Extract(0, psk.key)
    c_early_ts = Derive_Secret(early_secret, 'c e traffic', ch_hash)
    key, iv = derive_aead_key_iv(c_early_ts)
    plaintext = AEAD_Decrypt(key, iv, seq=0, ciphertext=ciphertext)

    # 防重放：单次使用票据是常见策略，但仍有窗口期限制
    if psk_store.is_used(psk_id):
        return drop_early_data()          # 相同票据被重放
    psk_store.mark_used(psk_id)

    # 应用层防重放：业务必须自行保证幂等
    if not is_idempotent(plaintext.method, plaintext.path):
        return reject_early_data_http_425_or_retry_without_early_data()
    deliver(plaintext)
```

### 4. 常见误区与进阶思考
常见误区：
1. 认为 TLS 1.3 的 0-RTT 数据有前向保密。实际上 early data 密钥来自静态 PSK，PSK 一旦泄露，历史 0-RTT 内容可被离线解密；只有后续握手完成后切换到 (EC)DHE 派生的 application traffic secret 才具备前向保密。
2. 认为服务器启用单次使用票据或限制 ticket age 就能完全阻止重放。单次使用票据存在窗口期：攻击者可在原始请求被处理前抢先重放；ticket age 只能限制时间窗口但不能阻止窗口内重放；跨服务器共享 PSK 时，各服务器独立标记使用状态，重放仍可能成功。
进阶思考题：假设你设计一个 HTTP JSON API，并允许在 TLS 1.3 0-RTT 中发送请求。请从 HTTP 方法语义、资源状态转换、以及服务器票据策略三个维度，给出一个可以安全启用 0-RTT 的判定条件集；并解释为什么即使启用单次使用票据，POST /orders 仍然不安全，而 GET /config 通常可以接受。
