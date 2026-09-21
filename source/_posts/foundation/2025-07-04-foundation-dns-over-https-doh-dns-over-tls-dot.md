---
title: "每日基础技术总结 · 2025-07-04 · DNS over HTTPS（DoH）与 DNS over TLS（DoT）"
date: 2025-07-04 20:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2025-07-04 · DNS over HTTPS（DoH）与 DNS over TLS（DoT）

## 📚 今日主题

> **DNS over HTTPS（DoH）与 DNS over TLS（DoT）**（网络基础）

### 1. 核心概念速览
DNS over HTTPS (DoH) 与 DNS over TLS (DoT) 是两种用于加密传统 DNS 查询的传输层协议，旨在解决明文 DNS 协议（UDP/53）面临的窃听、篡改和中间人攻击（MITM）问题。本质区别在于应用层协议的不同：DoH 将 DNS 消息封装在 HTTP/HTTPS 请求中，复用现有的 Web 基础设施（端口 443），利用 TLS 1.3+ 的加密特性；DoT 则在标准 TCP 端口 853 上直接运行 TLS 隧道封装原始 DNS 消息（RR）。它们位于 OSI 模型的应用层之上，但依赖于传输层的 TLS 握手机制，是整个网络安全体系从‘信任网络’转向‘零信任’架构的关键一环。对于全栈工程师而言，理解 DoH/DoT 是掌握现代 API 安全、CDN 边缘计算路由优化以及规避国家/企业级网络审查策略的前提，也是构建高可用、高安全后端微服务时处理内部服务发现（Service Discovery）的基础背景知识。

### 2. 底层原理剖析
1. 传输封装机制差异：
   - DoT: `Connection -> TLS Handshake -> [DNS Message Payload]`
     DNS 数据包直接被 TLS 记录层包裹。服务端监听 TCP 853 端口。优势是延迟低（无需 HTTP 头部开销），劣势是难以穿透防火墙（因其使用非标准端口且指纹明显）。
   - DoH: `Request -> HTTP/HTTPS Header + Body -> [JSON/XML/DNS Wire Format]`
     DNS 查询被序列化为 JSON 或二进制格式，嵌入 HTTPS POST/GET 请求体。服务端监听 TCP 443 端口。优势是与现有 HTTPS 基础设施无缝兼容，难以被深度包检测（DPI）区分，但增加了序列化/反序列化开销。
2. TLS 协商过程：两者均依赖 X.509 证书验证服务器身份，确保密钥交换和对称加密通道的建立。区别在于 DoH 可利用 ALPN (Application-Layer Protocol Negotiation) 明确标识为 'doth' 或直接作为普通 HTTPS 流量（增强隐蔽性）。
3. 对比前端概念：类比为 WebSocket vs HTTP Long Polling。DoT 类似 WebSocket，建立长连接后传输裸数据帧，效率高但协议特定；DoH 类似 HTTP Long Polling 或 GraphQL，通过通用的 HTTP 协议承载自定义业务数据，兼容性极佳但载荷冗余。前端开发中遇到的 CORS 预检（Preflight）在 DoH 中同样存在，因为它是基于 RESTful 风格的 GET/POST 请求。

### 3. 基础代码与实战验证
```text
// Python 示例：使用 ssl 模块手动建立 DoT 连接
import socket
import ssl

# 定义目标 DNS 服务器 (Google Public DNS via DoT)
doh_host = 'dns.google'
dot_port = 853

# 创建 TCP Socket
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# 包装 SSL Context，启用严格的证书验证
context = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
context.check_hostname = True
context.verify_mode = ssl.CERT_REQUIRED

# 执行 TLS 握手，此时才真正开始加密通信
secure_sock = context.wrap_socket(sock, server_hostname=doh_host)

try:
    # 手动构造 DNS 查询报文 (简化版，实际需遵循 RFC 1035 格式)
    # 这里仅演示如何发送加密后的字节流
    dns_query_bytes = b'\x12\x34\x01\x00\x00\x01\x00\x00\x00\x00\x00\x00\x05\x67\x6f\x6f\x67\x6c\x65\x03\x63\x6f\x6d\x00\x00\x01\x00\x01'
    
    # 发送数据，TLS 层负责将其分片并加密传输
    secure_sock.sendall(dns_query_bytes)
    
    # 接收响应，TLS 层负责重组解密
    response = secure_sock.recv(1024)
    print(f"Encrypted Response Length: {len(response)}")
finally:
    secure_sock.close()

// Node.js 伪代码：DoH 实现核心逻辑
const https = require('https');
const querystring = require('url');

// DoH 请求本质上是一个标准的 HTTPS POST 请求
const options = {
  hostname: 'dns.google',
  path: '/resolve?name=example.com&type=A', // Google DoH API 端点
  method: 'GET',
  headers: {
    'Accept': 'application/dns-json' // 关键：指定内容类型，告知服务器返回格式
  }
};

const req = https.request(options, (res) => {
  let data = '';
  res.on('data', chunk => data += chunk);
  res.on('end', () => {
    // 解析响应，通常是 JSON 格式，包含 Answer 段
    console.log(JSON.parse(data));
  });
});
req.end();
```

### 4. 常见误区与进阶思考
1. 认知误区：认为 DoH/DoT 能完全保证匿名性和隐私安全。实际上，DoH 只能防止 ISP 或本地路由器窥探查询内容，但 DoH 提供商（如 Cloudflare, Google）依然能看到你的真实 IP 和查询记录。此外，由于 DoH 伪装成普通 HTTPS 流量，企业防火墙可能允许其通过，导致内部 DLP（数据丢失防护）系统失效，这是企业网管最头疼的问题。
2. 进阶思考题：在 Kubernetes 集群内部，如果 Service Mesh（如 Istio）启用了 mTLS（双向 TLS），而上游 DNS 请求被强制重定向到集群外的 DoH 服务（如通过 Sidecar 代理），请分析这种架构下‘端到端加密’的定义范围是否发生了变化？这对服务间的信任边界（Trust Boundary）建模有何影响？
