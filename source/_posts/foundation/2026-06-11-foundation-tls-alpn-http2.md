---
title: "每日基础技术总结 · 2026-06-11 · TLS ALPN 扩展与 HTTP/2 协商"
date: 2026-06-11 08:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-06-11 · TLS ALPN 扩展与 HTTP/2 协商

## 📚 今日主题

> **TLS ALPN 扩展与 HTTP/2 协商**（网络基础）

### 1. 核心概念速览
核心概念速览：
- 定义：ALPN（Application-Layer Protocol Negotiation，应用层协议协商）是 TLS 扩展（RFC 7301），在握手期间由客户端声明其支持的应用层协议列表，服务端从列表中选择一个作为后续连接的应用协议。
- 本质：将应用层协议的选择从应用层（如 HTTP Upgrade）下沉到 TLS 握手层，加密通道建立的同时完成协议协商，避免额外 RTT。
- 解决的问题：HTTP/2 需要 TLS，但 TLS 建立后双方必须确定使用 HTTP/2 帧。用 HTTP Upgrade 会浪费 RTT，且明文不安全。ALPN 在 ClientHello 携带协议列表，ServerHello 返回选定协议，零额外开销。
- 机制：ClientHello 中 ALPN 扩展为 ProtocolNameList；ServerHello 返回选中的单个协议名。无交集时服务器发送 no_application_protocol 致命警报并终止握手。
- 位置：TLS 记录层与应用层之间。HTTP/2（RFC 7540）强制要求 TLS 下使用 ALPN 协商 h2。
- 为什么必须掌握：现代 Web 性能优化和全栈配置（网关、负载均衡、微服务）都依赖 ALPN；排障时需理解抓包中的 ALPN 字段。

### 2. 底层原理剖析
底层原理剖析：
1. 线格式：ALPN 扩展类型为 16。ClientHello 中扩展数据为 ProtocolNameList，即 ProtocolName 的向量。每个 ProtocolName 是 1 字节长度 + 协议名（如 'h2'、'http/1.1'）。ServerHello 中扩展数据为单个 ProtocolName。
2. 协商流程（TLS 1.2 示例）：
   a. 客户端构造 ClientHello，插入 ALPN 扩展，列表按客户端偏好排序。
   b. 服务器收到后，根据自身支持列表和策略选择其一。
   c. 服务器在 ServerHello 中返回选中的协议名。
   d. 客户端校验返回项是否在自己的列表中，否则中止。
   e. 之后双方使用该协议通信。
3. TLS 1.3 差异：ALPN 仍位于 ClientHello/ServerHello，明文传输，可被中间人观测；但 ServerHello 之后的扩展可加密，ALPN 本身不保密。
4. 与前端概念对比：前端内容协商（如 HTTP Accept 头）与 ALPN 类似，但 ALPN 在 TLS 层且是连接级，不可由应用层变更。Java 接口与 TS 接口的区别在于编译期 vs 运行时，ALPN 是运行时协议栈行为，类似前端特性检测，但更底层且对应用透明。异同点：相同处都是通过能力声明选择执行路径；不同处是 ALPN 发生在 TLS 内部，业务代码无法直接干预，而前端特性检测由业务代码主动执行。

### 3. 基础代码与实战验证
```text
基础代码（Node.js 内置 tls 模块，验证 ALPN 协商）：

const tls = require('tls');
const fs = require('fs');

// 服务器：设置 ALPN 支持列表
const options = {
  key: fs.readFileSync('key.pem'),   // 私钥
  cert: fs.readFileSync('cert.pem'), // 证书
  ALPNProtocols: ['h2', 'http/1.1']  // 按优先级列出服务器支持的协议
};

const server = tls.createServer(options, (socket) => {
  // socket.alpnProtocol 在 TLS 握手完成后被填充
  console.log('server: negotiated protocol =', socket.alpnProtocol);
  socket.end('ok');
});

server.listen(8443, () => {
  const client = tls.connect({
    port: 8443,
    host: '127.0.0.1',
    // 客户端声明支持的协议列表，顺序为偏好顺序
    ALPNProtocols: ['http/1.1', 'h2']
  }, () => {
    console.log('client: negotiated protocol =', client.alpnProtocol);
    client.end();
  });
});

关键注释：
- ALPNProtocols 数组在握手时被编码为 ProtocolNameList，随 ClientHello/ServerHello 传输。
- 协商结果保存在 socket.alpnProtocol / client.alpnProtocol 中。
- 若双方列表无交集，Node.js 会因 no_application_protocol 警报而中止连接（这里未捕获错误）。
- 运行前需生成证书：openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -days 365 -nodes
```

### 4. 常见误区与进阶思考
常见误区与进阶思考：

误区1：认为 ALPN 是 HTTP/2 专属。实际上 ALPN 是通用 TLS 扩展，可协商任意应用协议（如 h2、http/1.1、mqtt 等），HTTP/2 只是强制要求使用它。

误区2：混淆 ALPN 与 NPN。NPN 是 Google 早期方案，服务器发送列表、客户端选择；ALPN 是客户端发送列表、服务器选择。方向相反，NPN 已废弃。

思考题：客户端发送 ALPNProtocols: ['h2']，服务器只配置 ALPNProtocols: ['http/1.1']，TLS 握手结果是什么？为什么？

答案：服务器发送 no_application_protocol 致命警报，握手终止。因为 ALPN 要求服务器从客户端列表中选择，若没有交集则无法确定应用层协议，为免后续歧义，标准规定必须终止握手。注意，这不同于 HTTP 层的内容协商（如 Accept 头可以回退），ALPN 不自动回退。
