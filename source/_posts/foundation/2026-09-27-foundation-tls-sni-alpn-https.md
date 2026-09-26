---
title: "每日基础技术总结 · 2026-09-27 · TLS 的 SNI 与 ALPN 扩展在 HTTPS 中的角色"
date: 2026-09-27 07:03:48
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-27 · TLS 的 SNI 与 ALPN 扩展在 HTTPS 中的角色

## 📚 今日主题

> **TLS 的 SNI 与 ALPN 扩展在 HTTPS 中的角色**（网络基础）

### 1. 核心概念速览
SNI（Server Name Indication）与 ALPN（Application-Layer Protocol Negotiation）是 TLS 握手中的两个扩展字段，分别用于解决虚拟主机场景下的证书选择问题和应用层协议协商问题。SNI 本质上是 ClientHello 中携带的明文服务器域名，使服务端能在 TLS 握手阶段就确定该连接对应的证书与安全策略，从而支撑同一 IP 上部署多个 HTTPS 站点。ALPN 本质上是客户端在 ClientHello 中列出其支持的应用层协议（如 h2、http/1.1），服务端在 ServerHello 中选定一个，从而避免额外的 HTTP/2 升级往返（如 Upgrade 头或 h2c）。两者共同决定了 TLS 连接在握手完成后为何能够匹配正确的证书并直接运行预期的应用协议。它们位于 TLS 记录层与握手层之间，是 HTTPS 性能与安全的关键前置条件。专业工程师必须掌握它们是因其直接关系到 CDN 回源、反向代理路由、双栈迁移与 HTTP/2 部署的正确性，且 SNI 的明文特性本身又涉及安全的边界认知。

### 2. 底层原理剖析
TLS 握手由 ClientHello 开始，其中 Extension 字段携带 SNI 与 ALPN。SNI 的格式为 server_name 扩展，包含 name_type（host_name）与 host_name 字节串。其本质是证书选择器的索引：服务端在收到 ClientHello 后，尚未发送 ServerHello 之前，必须根据 SNI 查找对应的私钥与证书链；如果无法匹配，则返回 unrecongnized_name 告警或使用默认证书。该机制依赖 DNS 与 TLS 的分离：TLS 拿到的是应用层域名，而 IP 层与 TCP 层均不能提供该信息，因此没有 SNI 则虚拟主机无法做到多证书共存。ALPN 的格式为 ProtocolNameList，客户端按优先级排列协议标识符。服务端在 ServerHello 的 ALPN 扩展中返回选定的协议；若服务端不支持任何客户端列出的协议，则可以不返回 ALPN 而继续 TLS 握手，由应用层决定是否报错，但 HTTP/2 强制要求 ALPN 协商成功。ALPN 本质上是一种带外协商，它在加密层之上、应用层之下完成协议选择，避免了在应用数据中做显式升级握手。与前端概念的对比：SNI 可类比 Host 头，但 Host 头在 HTTP 层且被加密，SNI 在 TLS 层且明文；两者都可能被篡改，但 SNI 必须在证书校验前使用，因此其真实性无法依赖加密保证。ALPN 可类比 TS 中的类型标注：TS 的接口在编译期确定契约，ALPN 在握手期确定协议；但 TS 接口不参与运行时决策，而 ALPN 的协商结果直接决定后续字节流的解析方式（帧结构 vs 文本行）。另一个逼真的对比是：ALPN 类似于前端路由的协商，但前者是服务端单方面最终决定，客户端只能提出候选。SNI 的底层运行机制可形式化为：
1. 客户端构造 ClientHello，包含 SNI 扩展（明文域名列表）与 ALPN 扩展（协议列表）。
2. 服务端解析 ClientHello，先提取 SNI，在证书存储中匹配；若无匹配则选择默认证书（可能触发证书警告）。
3. 服务端提取 ALPN，与自身支持列表求交集，按服务端策略选择最优协议。
4. 服务端构造 ServerHello，携带所选 ALPN（若有），并完成密钥交换与证书发送。
5. 客户端验证证书对应域名与 SNI 一致（证书校验的域名匹配基于 SNI，而非 HTTP Host）。
6. 握手完成后，双方直接按 ALPN 所选协议解析应用数据。若没有 ALPN，则默认按 HTTP/1.1 处理（在 HTTPS 上下文中）。

### 3. 基础代码与实战验证
以下为最小化模拟逻辑（伪代码，非完整实现），展示 SNI 与 ALPN 在握手中的决策顺序：

```
// 服务端伪代码：处理 ClientHello 中的扩展
function onClientHello(hello) {
    // 1. 提取 SNI
    const sniList = hello.extensions.find('server_name');
    const serverName = sniList ? sniList.host_name : null;

    // 2. 基于 SNI 选择证书
    let cert = certStore.get(serverName);
    if (!cert) {
        cert = defaultCert; // 若无匹配，回退默认证书
        log.warning('SNI mismatch, using default cert');
    }

    // 3. 提取 ALPN 列表，并与服务端支持的协议集合取交集
    const clientProtocols = hello.extensions.find('alpn').protocol_list; // 如 ['h2','http/1.1']
    const supported = service.supportedProtocols; // 如 ['h2','http/1.1']
    let selected = null;
    for (const protocol of clientProtocols) {
        if (supported.includes(protocol)) {
            selected = protocol; // 按客户端优先级选择
            break;
        }
    }

    // 4. 构造 ServerHello：如果 selected 非空，则 ALPN 扩展写回该协议
    const serverHello = new ServerHello();
    if (selected) {
        serverHello.extensions.add('alpn', { selected_protocol: selected });
    }
    serverHello.certificate = cert;
    return serverHello;
}

// 客户端伪代码：验证证书与 SNI 是否匹配
function verifyCertificate(cert, sni, alpnRef) {
    // 证书中的 SAN 必须包含 SNI 域名，否则校验失败
    if (!cert.san.includes(sni)) throw new Error('证书域名不匹配');
    // 握手完成后，所有上层报文按 alpnRef 解析；若未协商 h2，则不能使用帧格式
    console.log('TLS established with', sni, 'protocol =', alpnRef);
}
```

实际验证可通过 curl 观察握手细节：
> curl -v https://example.com --resolve example.com:443:1.2.3.4

输出中 `SSL connection using TLSv1.3 / AEAD-CHACHA20-POLY1305` 之后的 `ALPN: server accepted h2` 即展示 ALPN 决策结果。若使用 `--tls-max 1.2` 与 `--compressed` 也能从回显中看到 SNI 发送的域名。

### 4. 常见误区与进阶思考
误区一：认为 SNI 是可信任的身份依据。SNI 与 Host 头一样是客户端可任意声明的明文字段，攻击者可伪造 SNI 请求他人证书，但这不会导致解密，因为证书校验取决于客户端信任链与域名匹配，而不是服务端对 SNI 的信任。真正的身份依据是服务端私钥签名的证书与客户端信任锚。同理，不能因 SNI 缺失而拒绝所有连接（如旧设备没有 SNI），正确做法是提供默认证书或告警。误区二：混淆 ALPN 与 HTTP/2 协商的关系。ALPN 是 TLS 扩展，HTTP/2 标准规定 TLS 上必须使用 ALPN 协商 h2；但 ALPN 本身可以协商任何协议（如 HTTP/1.1、SPDY、QUIC 内部的 HTTP/3 不使用 TLS 的 ALPN，而是 QUIC 的 ALPN 扩展）。因此不要认为有 ALPN 就有 HTTP/2，只有在 ALPN 结果为 h2 时才是 HTTP/2。另一个常见误区是：认为 ALPN 的协议列表是服务端优先选择，实际上规范要求客户端按偏好顺序列出，服务端必须从其中选择，但通常服务端会直接选第一个自己支持的，而不是按服务端优先级重排。思考题：一台服务器上同时运行多个 HTTPS 站点且共享同一 IP 和同一证书（证书的 SAN 包含多个域名），此时 SNI 是否仍然必要？如果所有站点的证书完全相同，SNI 是否仍然影响 TSL 握手过程（例如会话复用、TLS1.3 的 0-RTT、OCSP stapling）？请结合底层机制给出分析。
