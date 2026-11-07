---
title: "每日基础技术总结 · 2026-11-08 · DNS SPDZ扩展（如DoH/DoT）对中间人攻击的防御能力与隐私泄露边界"
date: 2026-11-08 07:01:21
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-08 · DNS SPDZ扩展（如DoH/DoT）对中间人攻击的防御能力与隐私泄露边界

## 📚 今日主题

> **DNS SPDZ扩展（如DoH/DoT）对中间人攻击的防御能力与隐私泄露边界**（网络基础）

### 1. 核心概念速览
## 核心概念速览

**术语校正**：题干中的「SPDZ」并非 DNS 相关术语（SPDZ 是安全多方计算/秘密共享协议族），这里应当理解为「DNS 加密传输扩展」，具体包括 DoT（RFC 7858/8310）、DoH（RFC 8484）、DoQ（RFC 9250，基于 QUIC）以及配套的 EDNS(0) Padding（RFC 7830/8467）。

**严谨定义**

- **DoT (DNS over TLS)**：在 TCP:853 上承载**未修改的 DNS wire format**，在每条 DNS 报文前附加 2 字节大端长度前缀（因为 TCP 是字节流、无消息边界），由 TLS 1.2+/1.3 提供机密性、完整性与服务器身份认证，ALPN 通常协商为 `dot`。
- **DoH (DNS over HTTP)**：将**同一份 wire format** 作为 HTTP 消息体传输，媒体类型固定为 `application/dns-message`；POST 直接放 body，GET 用 base64url 编码放在 `?dns=` 查询串。走 HTTPS(443)，复用 HTTP/2 或 HTTP/3 的多路复用与缓存语义。

**本质**：DNS 的**报文语法与语义完全不变**，把传输载体（carrier）从「明文、无认证、可被路径上任意节点读写的 UDP/53」替换为「经 PKI 认证的加密信道」。它只改变 **stub resolver → 递归解析器** 这一跳的威胁模型，不改变 DNS 体系固有的信任结构（权威链、递归解析器的可见性、PKI 的信任根）。

**解决的问题**

1. 路径被动嗅探导致 QNAME（即用户行为指纹）泄露；
2. 路径主动篡改：响应注入、伪造应答竞速、劫持与重定向、基于 53 端口的策略性阻断；
3. 使 DNS 解析具备「可认证的对端」，把「谁回答了我」从 IP 层假设升级为证书层证明。

**在体系中的位置**：应用层协议 DNS 的**传输加固层**——位于 TLS/QUIC 之上、DNS 消息语法之下。它直接决定后端服务的 egress 行为（容器 `resolv.conf`、K8s CoreDNS forward、Sidecar）、CDN 就近调度（ECS 的取舍）、可观测性埋点的迁移（从网络设备旁路转到解析器侧日志），以及隐私合规边界（QNAME + 源 IP 在 GDPR 下属个人数据）。

**为什么必须掌握**：它是「安全边界如何被重新划分」的典型案例——加密不是消除信任，而是**把信任从链路上的任意节点收敛到解析器运营方 + CA**；不理解这一点，就会把 DoH/DoT 误当成匿名化方案，在架构评审中做出错误的威胁建模。

### 2. 底层原理剖析
## 底层原理剖析

### 1. 载荷与信道解耦

```text
[ DNS Header 12B | QNAME | QTYPE 2B | QCLASS 2B | (Answer RRs...) ]   <- 恒定不变的 wire format
        |                                    |
   DoT: [len 2B][wire]  over TLS/TCP:853      DoH: HTTP body (application/dns-message) over TLS/TCP:443
```

关键推论：**报文 ID（Header 前 2 字节）在加密信道下已无安全意义**——它原本用于在无连接的 UDP 上匹配请求/响应并抵御伪造应答，TLS 的完整性与 TCP 的顺序性已覆盖该职责，因此 DoH 实践中常置 0 并依赖 HTTP 请求/响应配对。

### 2. DoT 的帧化与连接模型

- TCP 无消息边界 → 必须有 2 字节大端长度前缀；一次 `read` 可能读到半帧或多帧，必须做**流式重组**。
- 长连接 + 流水线（RFC 7766）：同一 TLS 连接上可并发多个 outstanding query，靠 ID 或响应顺序匹配；这带来**关联性泄露**（同一连接上的多次查询在路径上可被归并为同一客户端）。
- 部署模式两种（RFC 8310）：**机会模式（opportunistic）** 校验失败可回退明文，**严格模式（strict）** 失败即失败。防御强度差异完全由该配置决定。

### 3. DoH 的 HTTP 语义继承

- 请求/响应天然一一对应，无需 ID 匹配；
- 继承 HTTP 缓存：`Cache-Control: max-age` 由 DNS TTL 派生，于是**解析结果可被 HTTP 中间层缓存**——性能提升，但同时把「某客户端查询了某域名」这一事实暴露给 HTTP 缓存/CDN 层（新的泄露面）；
- 继承多路复用：多条查询共享一条 TLS 连接，牺牲 per-query 可分离性，但连接建立时刻、响应字节数、时序仍构成指纹；
- 继承中间盒可见性：URL 路径 `/dns-query` 与 TLS SNI（未启用 ECH 时）在路径上明文可见。

### 4. TLS 提供的三元组及其边界

机密性、完整性、**服务器认证**（PKI）。注意第三条的语义：它防的是「**假冒**解析器」，不防「解析器**作恶或留档**」。信任锚仍是系统 CA 库——企业 MITM 设备通过注入根证书即可合法解密全部 DoH 流量，这在工程上是「配置」而非「攻击」。

### 5. 防御矩阵（逐条对照）

| 威胁 | DoT/DoH 是否防御 | 说明 |
|---|---|---|
| 路径被动嗅探 QNAME | 是 | 但 SNI、目的 IP、包长/时序仍泄露 |
| 路径主动注入/篡改/劫持 | 是 | 依赖证书校验开启 |
| 降级到明文 53 | 仅严格模式 | 机会模式与 bootstrap 阶段是降级面 |
| 递归解析器记录/画像 | **否** | 信任转移而非消除 |
| CA 被攻破 / 企业根证书 | 否 | PKI 固有边界 |
| 递归→权威段（通常仍为明文 53） | 否 | 全路径加密需权威侧支持 DoT/DoQ，当前普及度低 |
| 流量分析（时序/体积/频次） | 部分 | Padding 缓解长度特征，时序特征不变 |
| EDNS Client Subnet 泄露子网 | 否 | ECS 会把客户端子网明文带给权威，隐私与 CDN 就近性直接冲突 |

### 6. 隐私泄露面清单（工程上必须逐项审计）

源/目的 IP、目的端口、TLS SNI（无 ECH 时）、TLS 握手特征与证书链（CT 日志可反查 DoH 主机名）、报文长度与填充策略、查询时序与频次、连接复用带来的查询归并、bootstrap 阶段的明文解析或硬编码 IP、解析器侧日志与留存策略、HTTP 层缓存命中/未命中（响应大小差异可反推共享度）。

### 7. 与前端已有体系的对照

- **DoT ≈ 手工分帧的二进制长连接**：等价于 `net.connect` 后自己维护 `[len][payload]` 帧头，形似 WebSocket 帧但只有长度字段、无 opcode/mask；独占端口，天然可被端口级策略识别与阻断。
- **DoH ≈ 一个 REST 端点**：语义上就是 `fetch('https://resolver/dns-query', {method:'POST', headers:{'content-type':'application/dns-message'}, body: arrayBuffer})`。因此它继承前端世界的一切 HTTP 中间件（代理、缓存、鉴权、WAF），也继承其全部可见性。
- **与普通 HTTPS 的差异**：普通 HTTPS 保护的是「资源内容」，而 DoH 保护的是「你要访问哪个域名」这一元数据——而 HTTPS 自身的 SNI/证书链恰恰泄露同一信息。所以 DoH 的隐私收益会被 TLS 握手抵消，除非叠加 ECH（RFC 9180 / ESNI）。这也是「HTTPS 已加密所以 DNS 无所谓」这一说法错误的原因：加密的是载荷，元数据依然裸奔。
- **与 Service Worker fetch 拦截的结构同构**：DoH 把解析权从系统 stub resolver 上移到应用层（浏览器/运行时），应用从而拥有独立解析策略；副作用是绕过 OS/网络侧的统一策略（split-horizon、内网私有域、家长控制、企业过滤），这正是企业环境 DoH 争议的根因。

### 3. 基础代码与实战验证
```text
// dns-dot-doh.js —— 仅用 Node.js 内置模块，直接验证两种 carrier 的底层差异
const tls = require('node:tls');
const https = require('node:https');
const dgram = require('node:dgram');

// 1) 手工构造 DNS wire format：12B 头部 + QNAME + QTYPE/QCLASS
function buildQuery(name, type = 1 /* A */) {
  const header = Buffer.alloc(12);
  header.writeUInt16BE(0x1234, 0); // ID：加密信道下仅用于配对，可置 0
  header.writeUInt16BE(0x0100, 2); // Flags：RD=1 期望递归
  header.writeUInt16BE(1, 4);      // QDCOUNT=1；ARCOUNT=0 即不带 EDNS(0)，故无 OPT/Padding
  const labels = name.split('.').map(l => {
    const b = Buffer.from(l, 'ascii');
    return Buffer.concat([Buffer.from([b.length]), b]); // 每标签前置 1 字节长度（上限 63）
  });
  const qname = Buffer.concat([...labels, Buffer.from([0])]); // 根标签 0x00 终止
  const tail = Buffer.alloc(4);
  tail.writeUInt16BE(type, 0);    // QTYPE
  tail.writeUInt16BE(1, 2);       // QCLASS = IN
  return Buffer.concat([header, qname, tail]);
}

// 2) 解析 Answer Section，返回 A 记录
function parseA(msg) {
  let off = 12;
  while (msg[off] !== 0) off += msg[off] + 1;  // 跳过 QNAME 标签序列
  off += 1 + 4;                                 // 根标签 + QTYPE/QCLASS
  const ancount = msg.readUInt16BE(6);
  const out = [];
  for (let i = 0; i < ancount; i++) {
    if ((msg[off] & 0xc0) === 0xc0) off += 2;   // 压缩指针 0b11xxxxxx：指向别处的域名，占 2 字节
    else { while (msg[off] !== 0) off += msg[off] + 1; off += 1; }
    const rtype = msg.readUInt16BE(off);
    off += 8;                                   // 跳过 TYPE(2)+CLASS(2)+TTL(4)
    const rdlen = msg.readUInt16BE(off); off += 2;
    if (rtype === 1 && rdlen === 4) out.push([...msg.subarray(off, off + 4)].join('.'));
    off += rdlen;                               // RDLENGTH 决定 RDATA 长度，逐 RR 前进
  }
  return out;
}

// 3) 对照基线：明文 UDP/53 —— QNAME 在链路上逐字节可读
function queryPlain(server, wire) {
  return new Promise((resolve, reject) => {
    const s = dgram.createSocket('udp4');
    s.once('message', m => { s.close(); resolve(m); });
    s.once('error', reject);
    s.send(wire, 53, server);                   // 无加密无认证：任何 on-path 节点可读可改
  });
}

// 4) DoT：TLS 之上加 2 字节长度前缀，流式重组
function queryDoT(sni, ip, wire) {
  return new Promise((resolve, reject) => {
    // servername 必须显式设置：连的是 IP，但证书校验必须按主机名进行——这是抗 MITM 的唯一依据
    const sock = tls.connect({ host: ip, port: 853, servername: sni, ALPNProtocols: ['dot'] }, () => {
      const frame = Buffer.alloc(2 + wire.length);
      frame.writeUInt16BE(wire.length, 0);      // RFC 7858：2 字节大端长度前缀，补救 TCP 无消息边界
      wire.copy(frame, 2);
      sock.write(frame);
    });
    let buf = Buffer.alloc(0);
    sock.on('data', chunk => {
      buf = Buffer.concat([buf, chunk]);
      if (buf.length < 2) return;
      const len = buf.readUInt16BE(0);
      if (buf.length < 2 + len) return;         // 一帧可能跨多个 TCP segment，必须累积
      resolve(buf.subarray(2, 2 + len));
      sock.end();
    });
    sock.on('error', reject);                   // 含证书校验失败：严格模式下应直接失败而非回退
  });
}

// 5) DoH：wire format 作为 HTTP body，语义等价于一个二进制 REST 端点
function queryDoH(url, wire) {
  return new Promise((resolve, reject) => {
    const req = https.request(url, {
      method: 'POST',
      headers: {
        'content-type': 'application/dns-message', // RFC 8484 唯一媒体类型：body 即原始 DNS 报文
        'accept': 'application/dns-message',
        'content-length': wire.length,
      },
    }, res => {
      const chunks = [];
      res.on('data', c => chunks.push(c));
      res.on('end', () => {
        if (res.statusCode !== 200) return reject(new Error('HTTP ' + res.statusCode));
        resolve(Buffer.concat(chunks));         // TTL 由 Cache-Control: max-age 表达，可被 HTTP 中间层缓存
      });
    });
    req.on('error', reject);
    req.end(wire);
  });
}

(async () => {
  const wire = buildQuery('example.com');
  console.log('plain UDP:', parseA(await queryPlain('8.8.8.8', wire)));
  const dot = await queryDoT('cloudflare-dns.com', '1.1.1.1', wire);
  const doh = await queryDoH('https://cloudflare-dns.com/dns-query', wire);
  console.log('DoT      :', parseA(dot));  // 两条路径载荷字节结构完全一致，差异仅在 carrier
  console.log('DoH      :', parseA(doh));
})();

// 验证命令：tcpdump -i any -X 'port 53' 可看到 example.com 的 ASCII；
//           tcpdump -i any 'port 853' 只能看到 TLS record，QNAME 不可见，但 SNI=cloudflare-dns.com 可见。
```

### 4. 常见误区与进阶思考
## 常见误区与进阶思考

### 误区一：把 DoH/DoT 当成「DNS 端到端加密 + 匿名化」

它只加密 **stub → 递归解析器** 一跳。递归解析器仍看到完整 QNAME + 客户端 IP 并可长期留存画像；递归 → 权威段绝大多数仍是明文 UDP/53。真正的变化是**信任位置迁移**：从「链路上任意节点」收敛到「解析器运营方 + CA 信任根」。因此：
- 更换解析器运营方 = 更换信任对象，不是取消信任；
- 企业根证书注入或 CA 失陷时，加密完全失效，且这种失效在**配置层是合法的**；
- 未启用严格模式时，证书失败回退明文 53 构成降级面；
- 未启用 ECH 时，TLS SNI（以及 DoH 的 URL 路径 `/dns-query`）在路径上明文可见，**DoH 想保护的元数据恰好被承载它的 TLS 泄露**。

### 误区二：混淆 DNSSEC 与 DoH/DoT 的防护目标

- DNSSEC（RFC 4033-4035）提供**数据来源认证与完整性**，是对抗伪造应答的密码学机制，但**不提供任何机密性**——查询与应答全明文，且 NSEC/NSEC3 的否定证明会暴露区内容枚举；
- DoH/DoT 提供**传输机密性与信道级完整性**，但**不保证应答内容真实**（一个作恶的递归解析器可以返回任意 A 记录，而 TLS 依然校验通过）。

二者正交、互补，不可互相替代。仅部署 DoH 而不部署 DNSSEC，等于「防住了路人，没防住信使」。另一个衍生误区：认为 DoH 会破坏 `NXDOMAIN` 语义或使 split-horizon 失效——真正的原因是应用层解析器（浏览器自带 DoH）**绕过了系统 stub resolver 的搜索域与转发策略**，是策略越权而非协议缺陷。

### 进阶思考题

假设攻击者在路径上，**只能丢包、延迟与观察，不能篡改内容**。在 DoH 场景下：

1. HTTP/2 多路复用把多条查询绑定到同一条 TLS 连接，它削弱了哪一类指纹（per-query 的包长/时序），同时又引入了哪种新的可观测面（连接建立时刻、流水线深度、响应字节数差异、连接存活期与查询频次的相关性）？
2. EDNS(0) Padding（RFC 8467）按块长对齐消除长度特征，但它对**时序侧信道**毫无作用；若目标域名集合有限（例如只区分 10 个候选站点），攻击者能否仅靠「查询发起时刻 + 响应到达时刻 + 是否命中递归缓存」的联合分布完成分类？请给出可测量的特征向量与判别思路。
3. 进一步：如果要求同时满足「抵抗上述流量分析」与「不破坏 CDN 就近调度」，Padding 策略、连接复用粒度与 ECS 的开关应如何协同设计？这三点之间存在什么不可同时满足的约束？
