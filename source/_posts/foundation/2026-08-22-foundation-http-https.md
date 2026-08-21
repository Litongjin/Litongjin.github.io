---
title: "每日基础技术总结 · 2026-08-22 · HTTP/HTTPS 握手与加密过程"
date: 2026-08-22 07:00:49
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-22 · HTTP/HTTPS 握手与加密过程

## 📚 今日主题

> **HTTP/HTTPS 握手与加密过程**（前端底层与计算机基础）

### 1. 核心概念速览
HTTP 是应用层无状态请求-响应协议，本质是客户端与服务器之间基于 TCP 的文本/二进制帧交换。HTTPS 并非独立协议，而是 HTTP over TLS，即在 TCP 之上增加 TLS 安全层。握手指建立可靠连接和加密参数协商的过程：HTTP 的握手是 TCP 三次握手（SYN/SYN-ACK/ACK）以建立可靠字节流；HTTPS 额外包含 TLS 握手，完成身份认证、密钥协商和加密参数确认。它解决三方面问题：机密性（防止窃听）、完整性（防篡改）、身份认证（防伪装）。在计算机体系中，HTTP/HTTPS 位于 TCP/IP 协议栈的应用层，直接支撑 Web、REST API 和云原生架构。专业工程师必须掌握握手过程，因为性能优化（连接复用、TLS 握手延迟）、安全配置（证书链、加密套件）以及故障排查（握手失败、抓包分析）均依赖对底层机制的精确理解。

### 2. 底层原理剖析
握手分为两个层次：TCP 握手与 TLS 握手。
1. TCP 三次握手（传输层）：客户端发送 SYN（seq=x），服务器回应 SYN-ACK（seq=y, ack=x+1），客户端发送 ACK（seq=x+1, ack=y+1）。本质是双方同步初始序列号，建立双向可用的可靠连接。连接状态在两端各自维护，由内核协议栈完成。
2. TLS 握手（安全层）：以 TLS 1.2 为例，完整流程为：
   a. ClientHello：客户端生成随机数 client_random，携带支持的 TLS 版本、加密套件列表（如 ECDHE-RSA-AES128-GCM-SHA256）和扩展（SNI）。
   b. ServerHello：服务器选择加密套件和 TLS 版本，返回随机数 server_random，并下发证书（含公钥）和密钥交换参数（如 ECDHE 的服务器公钥）。
   c. 客户端验证证书链（根证书、中间证书、有效期、域名匹配），若有效则生成 pre-master secret（在 ECDHE 中为客户端公钥，在 RSA 中为随机数）。
   d. 双方通过 Key Schedule 算法，由 client_random、server_random 和 pre-master 共同派生出对称密钥（主密钥、工作密钥）。
   e. 客户端发送 ChangeCipherSpec（通知后续加密）和 Finished（加密的握手摘要），服务器同样响应，完成握手。
   后续应用数据使用对称加密（如 AES-GCM）并附加 MAC/认证标签，保证机密性和完整性。
3. TLS 1.3 优化：握手从 2-RTT 降为 1-RTT，ClientHello 直接携带客户端密钥共享参数（如 X25519），服务器可并行回复服务器参数和证书，减少了消息往返。
对比前端已有概念：HTTP 的 TCP 连接与 TLS 会话类似于 Java 接口与 TypeScript 接口的关系——前者是运行时内核维护的真实资源，后者是编译期类型约束；同样，TCP 握手是传输层的状态同步，而 TLS 握手是应用层的安全状态协商，两者不能混淆。类似地，前端中的 WebSocket 握手（HTTP Upgrade）只是协议升级，并非传输层连接。

### 3. 基础代码与实战验证
```text
以下代码使用 Node.js 内置 tls 模块，建立 TCP 连接后自动执行 TLS 握手，并在 secureConnect 回调中输出协商结果。

    const tls = require('tls');
    const socket = tls.connect({
      host: 'example.com',
      port: 443,
      servername: 'example.com'   // SNI 扩展，让服务器返回对应虚拟主机证书
    }, () => {
      // secureConnect 回调触发，表示 TLS 握手已成功完成
      // 此时安全层已经协商出对称密钥，后续 socket 读写自动加解密
      console.log('协商出的 TLS 协议版本:', socket.getProtocol()); // 如 'TLSv1.3'
      console.log('加密套件:', socket.getCipher()); // 如 'TLS_AES_128_GCM_SHA256'
      // 发送 HTTP/1.1 请求，应用数据会被 TLS 加密
      socket.write('GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n');
    });
    socket.on('data', (data) => {
      console.log('解密后的响应:', data.toString());
      socket.end();
    });

若需观察握手消息，可执行 openssl s_client -connect example.com:443 -tls1_2 -msg，-msg 会打印每个握手记录的明文（包括 ClientHello、ServerHello、证书等）。

注意：socket.write 写入的数据会交给 TLS 记录层加密，再经 TCP 发送；data 事件收到的数据是 TLS 记录层解密后的明文。这就是 HTTPS 与 HTTP 在 API 层面的唯一差异——传输层之上多了一个加解密层。
```

### 4. 常见误区与进阶思考
常见误区 1：将 TCP 三次握手与 TLS 握手混为一谈，认为 HTTPS 握手就是 TCP 连接成功后顺带做一次加密协商。实际上 TCP 握手在传输层，由内核处理，主要交换 SYN 和 ACK 以同步序列号；TLS 握手在安全层，运行于 TCP 之上，包含 ClientHello/ServerHello/证书交换/密钥协商/Finished 等至少 4 条消息（TLS 1.2 为 2-RTT，TLS 1.3 为 1-RTT）。两者是完全独立的状态机，只是共享同一个 TCP 连接。

常见误区 2：认为 HTTPS 的数据传输使用非对称加密，因此性能开销大。实际上非对称加密（或 ECDHE 密钥交换）只用于握手阶段的身份认证和对称密钥协商；一旦对称密钥生成，后续所有 HTTP 消息都使用对称加密（如 AES-GCM）和 MAC 认证，对称加密在硬件和算法上远快于非对称加密。TLS 1.3 更是仅保留 ECDHE 等前向保密算法，完全移除了 RSA 密钥交换。

深度思考题：为什么 TLS 1.3 废除了 RSA 密钥交换（即客户端生成 pre-master 并用服务器公钥加密传输）？请从“前向保密”的角度解释：若攻击者长期记录密文并最终获得服务器私钥，RSA 交换能否解密历史通信？这反映了密钥协商中哪种安全属性？
