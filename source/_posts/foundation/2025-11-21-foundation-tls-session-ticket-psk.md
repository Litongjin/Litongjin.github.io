---
title: "每日基础技术总结 · 2025-11-21 · TLS 的会话恢复：Session Ticket 与预共享密钥（PSK）"
date: 2025-11-21 20:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2025-11-21 · TLS 的会话恢复：Session Ticket 与预共享密钥（PSK）

## 📚 今日主题

> **TLS 的会话恢复：Session Ticket 与预共享密钥（PSK）**（网络基础）

### 1. 核心概念速览
TLS 会话恢复（Session Resumption）旨在消除 TLS 1.3 之前的完整握手带来的 Round Trip Time (RTT) 开销。Session Ticket 是一种无状态恢复机制，服务端将加密的会话状态封装在 Ticket 中发给客户端，后续连接由携带 Ticket 的客户端直接证明身份，无需服务端查询缓存。预共享密钥（PSK，现称 External PSK 或 Early Data Context）是 TLS 1.3 引入的更强恢复机制，双方预先通过带外信道共享主密钥种子，握手时直接基于此建立主密钥，实现 0-RTT 数据传输。掌握其本质对于理解现代高性能 Web 协议优化、CDN 架构设计以及安全传输层状态管理至关重要，它是平衡安全性、性能与可扩展性的核心权衡点。

### 2. 底层原理剖析
Session Ticket 机制：1. 首次握手：服务端完成 Full Handshake，生成 Session Ticket Key (STK)。2. 票据创建：服务端使用 STK 对 Session State（包含主密钥、压缩算法、序列号等）进行对称加密，生成 EncryptedState。3. 票据分发：EncryptedState 作为 session_ticket 扩展字段发送给 ClientHello。4. 恢复握手：客户端在后续 ClientHello 中发送该 Ticket。5. 票据验证：服务端使用同一 STK 解密 EncryptedState，校验 MAC/Integrity，若成功则跳过密码学交换，直接导出新的主密钥，进入 1-RTT 模式。PSK 机制：1. 外部协商：通过非 TLS 通道（如 DHCP、DNS、或应用层协议）预先共享一个秘密值（PresharedKey）。2. 握手绑定：ClientHello 中携带 psk_key_exchange_modes 和 psk_identity。3. 密钥衍生：双方基于 PresharedKey 和握手过程中的随机数混合（mixing），推导出现场主密钥。4. 优势：彻底移除前向保密计算中的 Diffie-Hellman 部分，支持 0-RTT 数据（早期数据），但需警惕重放攻击（Replay Attacks）导致的敏感信息泄露风险。

### 3. 基础代码与实战验证
```text
// Node.js tls 模块验证 Session Ticket 机制
const tls = require('tls');
const fs = require('fs');

// 服务器端配置
const serverOpts = {
  key: fs.readFileSync('server-key.pem'),
  cert: fs.readFileSync('server-cert.pem'),
  // 关键：启用 session ticket，默认开启
  sessionTicketLifetime: 86400 
};

const server = tls.createServer(serverOpts, (socket) => {
  socket.setEncoding('utf-8');
  console.log(`Connected`);
  socket.write('Welcome\n');
});

server.listen(9999, () => {
  // 客户端第一次连接
  const client1 = tls.connect(9999, 'localhost', { rejectUnauthorized: false });
  client1.on('secure', () => {
    // 获取生成的 session ticket
    const ticket = client1.getSession();
    console.log('First Connection Ticket:', ticket ? 'Generated' : 'Null');
    client1.end();
  });

  // 模拟稍后时间的第二次连接，携带之前保存的 ticket
  setTimeout(() => {
    const client2 = tls.connect(9999, 'localhost', { rejectUnauthorized: false });
    // 手动注入之前获取的 ticket 以模拟恢复
    if(ticket) client2.setSession(ticket);
    
    client2.on('secure', () => {
      console.log('Second Connection Established (Potential Resumption)');
      // 检查连接是否复用
      console.log('Session Reused:', client2.isSessionReused());
      client2.end();
    });
  }, 1000);
});
/* 底层逻辑说明：
 * setSession() 仅在客户端侧将 Ticket 放入 ClientHello。
 * isSessionReused() 返回 true 当且仅当服务端成功解析并信任了该 Ticket，
 * 未发生完整的 Diffie-Hellman 密钥交换，仅进行了密钥更新（Key Update）。 */
```

### 4. 常见误区与进阶思考
1. 混淆 Session ID 与 Session Ticket：Session ID 是有状态的，服务端必须维护映射表（内存/Redis），导致横向扩展困难；Session Ticket 是无状态的，服务端仅需持有加密密钥即可处理任何客户端的请求，适合大规模分布式部署。2. 忽视 0-RTT 的重放风险：PSK 允许 0-RTT 数据，但这部分数据没有正向保密性且可能被攻击者捕获后重放。工程师必须在应用层实现幂等性保护或为 0-RTT 数据设置极短的生存时间窗口，绝不能假设 0-RTT 数据的不可篡改性和唯一性。进阶思考：如果服务端丢失了 Session Ticket Key (STK)，已颁发的 Ticket 为何立即失效？反之，如果轮换了 STK，之前用旧密钥加密的 Ticket 会发生什么？这体现了无状态恢复中对‘密钥轮换（Key Rotation）’策略的依赖——通常采用多密钥并行验证期以平滑过渡。
