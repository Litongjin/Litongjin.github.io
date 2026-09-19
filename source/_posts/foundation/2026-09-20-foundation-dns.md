---
title: "每日基础技术总结 · 2026-09-20 · DNS 解析流程"
date: 2026-09-20 07:02:17
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-20 · DNS 解析流程

## 📚 今日主题

> **DNS 解析流程**（前端底层与计算机基础）

### 1. 核心概念速览
# 核心概念速览

- 定义：DNS（Domain Name System）是分层、分布式的命名系统与查询协议，核心功能是将域名映射为资源记录（RR），最常见为 A/AAAA 地址，也承载 NS、CNAME、MX、TXT、SRV、CAA、SOA 等。
- 本质：全球分布式数据库 + 委派授权体系 + 带 TTL 的多级缓存查询协议。它不维护单一中心映射，而是通过根、顶级域、权威服务器的层级委派完成名称到资源记录的解析。
- 解决什么问题：IP 地址可变、难记忆，且同一服务需要多地址、故障切换、地域调度、邮件路由、证书验证。DNS 提供稳定名称到动态资源记录的间接层。
- 机制：客户端 stub resolver 调用系统解析器；系统解析器可能查 hosts/缓存，然后向递归解析器发 RD=1 查询；递归解析器从根开始迭代查询 TLD、权威服务器，获取最终 RR，按 TTL 缓存并返回。
- 体系位置：应用层协议，依赖 UDP/TCP/IP。它发生在 TCP 连接、TLS 握手、HTTP 请求之前，是几乎所有网络通信的引导层。AI/分布式系统中，服务发现、K8s Service、模型服务网关、训练集群节点寻址同样依赖 DNS 或同类机制。
- 为什么必须掌握：DNS 直接决定首包延迟、连接建立、CDN 命中、灰度/故障切换、跨域与安全边界。只会看 HTTP 层无法定位 DNS 污染、缓存未过期、CNAME 链、DoH 绕过 hosts、DNS 泄露等问题。

### 2. 底层原理剖析
# 底层原理剖析

解析路径从近到远，命中即短路：

1. 应用/浏览器缓存：浏览器网络栈维护 DNS cache，键通常为 (name, type, class)，受 TTL 约束。
2. OS stub resolver：应用调用 getaddrinfo()，查询 /etc/hosts（Windows 为 hosts 文件），经 nsswitch.conf 选择解析后端；systemd-resolved、nscd、DNS Client 服务可能提供系统缓存。
3. 递归解析器：stub 向配置的递归解析器发查询，RD=1 表示请求递归，期望得到最终答案。
4. 迭代查询：递归解析器若无缓存，从根服务器开始，逐级向下：
   - 根返回 TLD 的 NS 记录及 glue A/AAAA；
   - TLD 返回权威域名服务器的 NS；
   - 权威服务器返回最终 RR，如 A/AAAA 或 CNAME。
5. 返回与缓存：递归解析器按 RR TTL 缓存，stub/OS/浏览器也可能缓存；负缓存由 SOA MINIMUM 控制。

精确伪代码：

function resolve(name, type):
    if appCache.hit(name, type) and not appCache.expired: return appCache.get
    if hosts.has(name): return hosts.get
    if osCache.hit(name, type) and not osCache.expired: return osCache.get
    resp = recursiveResolver.query(name, type, RD=1)
    appCache.put(resp, resp.TTL)
    return resp

function recursiveResolver.query(name, type):
    if resolverCache.hit(name, type) and not resolverCache.expired: return resolverCache.get
    server = rootServer
    loop:
        resp = server.query(name, type, RD=0)
        if resp.answer.has(type or CNAME):
            if resp.answer.CNAME: name = resp.answer.CNAME; continue
            resolverCache.put(resp, resp.TTL)
            return resp
        if resp.authority.NS:
            server = glueAddress(resp.authority.NS) or resolve(resp.authority.NS)
        if resp.truncated:
            retry over TCP

递归与迭代的区别：
- 递归查询：客户端 -> 递归解析器，RD=1，由解析器承担全部查询工作。
- 迭代查询：递归解析器 -> 各级权威，RD=0，权威只返回下一跳或答案。

传输与协议：
- 默认 UDP 53；响应 TC=1 截断时改用 TCP 53。EDNS0 允许更大 UDP 负载。
- DoT 将 DNS 封装于 TLS，端口 853；DoH 封装于 HTTPS，端口 443；DoQ 基于 QUIC。加密只保护传输，不校验数据真实性，真实性需 DNSSEC。
- 记录与行为：A/AAAA 为最终地址；CNAME 产生别名链，会触发额外解析；NS 表示委派；SOA 用于负缓存和区域权威。

与前端已有概念对比：
- 前端模块解析（import 'react' -> node_modules/react）与 DNS 都是名称到位置的解析，但模块解析在构建期/本地执行，算法确定，无分布式授权和 TTL；DNS 是运行时分布式协议，存在缓存一致性窗口和网络故障。
- 浏览器缓存层级（Memory Cache、Disk Cache、HTTP Cache）与 DNS 多级缓存类似，都是逐层查询、命中短路；但 DNS 缓存键为 (name, type, class)，TTL 由权威 RR 控制；HTTP 缓存受 Cache-Control、Expires、ETag 控制。
- Service Worker 可拦截 fetch，但无法拦截 DNS，因为 DNS 发生在 TCP/TLS 之前，SW 运行在 HTTP 语义层。

### 3. 基础代码与实战验证
```text
# 基础代码与实战验证

Node.js 内置 dns 模块可区分两条路径：dns.lookup 走 OS stub resolver，dns.resolve4 直接构造 DNS 查询。

const dns = require('dns');

// dns.lookup 调用 OS getaddrinfo，受 /etc/hosts、nsswitch.conf、系统缓存影响
dns.lookup('example.com', { all: true }, (err, addresses) => {
  if (err) throw err;
  console.log('lookup (OS stub):', addresses);
});

// dns.resolve4 使用 c-ares 直接向递归解析器发 A 查询，绕过 hosts 和部分系统缓存
dns.resolve4('example.com', (err, addresses) => {
  if (err) throw err;
  console.log('resolve4 (direct DNS):', addresses);
});

// 修改 resolve 使用的递归解析器，不影响 lookup 的 OS 路径
dns.setServers(['1.1.1.1', '8.8.8.8']);
dns.resolve4('example.com', (err, addresses) => {
  if (err) throw err;
  console.log('resolve4 via 1.1.1.1:', addresses);
});

验证迭代查询与递归查询的差异，可用 dig：
dig example.com          # 默认向本地递归解析器发 RD=1 查询
dig @8.8.8.8 example.com # 指定递归解析器
dig +trace example.com   # 从根开始执行迭代查询，观察根、TLD、权威的委派链

关键观察：若 /etc/hosts 中写入 example.com 的自定义 IP，dns.lookup 返回该 IP，而 dns.resolve4 仍返回真实 DNS 记录。这直接验证了 stub resolver 路径与直接 DNS 查询路径的分离。
```

### 4. 常见误区与进阶思考
# 常见误区与进阶思考

- 误区一：认为 DNS 解析是一次远程查询，忽略多级缓存和 TTL。实际浏览器、OS、递归解析器、权威服务器都可能缓存；修改记录后，旧值会持续到各级 TTL 到期。CNAME 链会放大查询次数和延迟。
- 误区二：把 DNS 与 HTTP 安全策略混为一谈。CORS、同源策略作用于 HTTP 响应和脚本访问，不能阻止 DNS 查询，也不能让 Service Worker 拦截 DNS。DoH/DoT 只加密 DNS 传输，不验证记录真实性；真实性需要 DNSSEC。另一个常见错误是认为 /etc/hosts 一定优先于所有 DNS：浏览器启用安全 DNS/DoH 时，可能绕过系统解析器和 hosts。

思考题：当你在生产环境修改 A 记录后，不同地区、不同运营商、不同浏览器看到新 IP 的时间不一致。请从 TTL、递归解析器缓存、浏览器/OS 缓存、CNAME 链、权威 NS 多实例一致性、Anycast/GeoDNS 等角度，设计一套定位与验证方案，并说明每一步应该使用什么工具观察哪一层缓存。
