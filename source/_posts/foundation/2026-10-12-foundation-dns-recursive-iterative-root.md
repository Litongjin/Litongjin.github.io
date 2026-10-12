---
title: "每日基础技术总结 · 2026-10-12 · DNS 正向解析流程：递归查询与迭代查询的区别及根服务器作用"
date: 2026-10-12 10:07:14
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-12 · DNS 正向解析流程：递归查询与迭代查询的区别及根服务器作用

## 📚 今日主题

> **DNS 正向解析流程：递归查询与迭代查询的区别及根服务器作用**（网络基础）

### 1. 核心概念速览
DNS 正向解析是将人类可读的完全限定域名（FQDN）映射为 IP 地址（A/AAAA 记录）及其他资源记录的过程。递归查询：客户端（stub resolver）向配置的递归解析器发起一次查询，要求获得最终答案；递归解析器负责完成后续全部查询并返回结果。本质是责任委托，客户端无需了解层级结构。迭代查询：DNS 服务器收到查询后，不代替查询者完成全部解析，而是返回下一步应查询的服务器引用（NS 及粘接记录）；查询方通过多轮请求逐步逼近最终答案。本质是逐步导航，查询方显式控制流程。根服务器是 DNS 层级最顶端的权威服务器集群，维护所有顶级域（TLD）的 NS 记录与粘接地址；它在迭代查询中充当解析入口，将查询引导至对应 TLD 服务器，但绝不直接返回最终域名 IP。该机制解决了全球分布式域名空间的可扩展、容错与授权问题。位于 TCP/IP 应用层，是 HTTP/TLS、微服务发现、CDN 调度、邮件路由等所有域名依赖服务的前置基础。专业工程师必须掌握它，因为调试解析故障、设计高可用服务发现、防范 DNS 劫持/缓存投毒、优化 CDN 与负载均衡都依赖对递归/迭代模式的精确理解。

### 2. 底层原理剖析
递归解析器收到客户端递归查询后，先查本地缓存（含 TTL），命中则直接返回。未命中时，从根提示文件（root hints，内置 13 个根服务器 IP）开始迭代查询：1. 向根服务器发送 example.com A 查询；2. 根服务器检查其 TLD 数据库，返回 com. TLD 的 NS 记录及对应粘接 A/AAAA 记录（referral）；3. 递归解析器缓存该 referral，选一个 TLD 服务器继续发送同一查询；4. TLD 服务器返回 example.com 的权威 NS 记录及粘接记录；5. 递归解析器再向权威服务器发送查询；6. 权威服务器根据区域文件返回最终 A 记录或 CNAME 链；7. 递归解析器缓存最终结果，构造响应返回给客户端。整个过程中，客户端只发出一次递归查询；递归解析器与其他服务器之间全部使用迭代查询（除非配置了转发器或上游递归）。

递归与迭代的本质区别：递归查询中，被查询方承担“负责到底”的义务，返回最终结果或错误；迭代查询中，被查询方只返回“更接近答案的服务器地址”，查询方自行继续。控制流不同：递归是客户端-服务器一对一请求/响应，迭代是查询方-多服务器循环请求/引用。

根服务器作用：根服务器不存储任何具体域名记录，只维护 TLD 授权表。递归解析器启动时通过根提示文件或内置列表获取根服务器地址，根提示是 DNS 解析的信任锚点。根服务器通过 Anycast 部署在全球多节点，提供低延迟和抗 DDoS。没有根服务器迭代查询无法启动（除非使用转发器或私有根）。

与前端的机制对比：递归查询与 BFF/GraphQL 聚合层在控制流上同构——客户端一次请求，服务端内部多次下游查询后返回最终结果；差异在于 DNS 递归发生在 DNS 协议内部，由 RFC 1034/1035 定义，而 BFF 是应用层 HTTP 聚合。迭代查询与前端手动编排多个 API 请求的循环模式对应——查询方根据每次响应中的“下一跳”信息决定下一次请求，类似于处理分页游标或 HATEOAS 的链接，但 DNS 迭代的“下一跳”由 NS 和粘接记录显式给出。

### 3. 基础代码与实战验证
```text
实战验证：使用 dig 的 +trace 选项模拟递归解析器的迭代过程，从根服务器开始逐步追踪。

命令：
  dig +trace example.com

输出关键段（以实际环境为准）：
  ... root servers ...
  com. ... NS ...
  example.com. ... NS ...
  example.com. ... A 93.184.216.34

该命令从根提示文件中的根服务器 IP 开始，逐级发送迭代查询，展示根→TLD→权威的完整路径。

极简伪代码（无框架，模拟递归解析器的迭代解析核心循环）：

  async function iterativeResolve(domain, qtype) {
    // 初始候选服务器为根提示中的根服务器地址
    let servers = rootHints;
    let currentName = domain;
    while (true) {
      // 向当前候选服务器发送迭代查询（底层为 UDP/TCP 端口 53 的 DNS 报文）
      const resp = await dnsQuery(servers[0], currentName, qtype);
      // 若响应包含目标类型的最终记录（如 A），则缓存并返回
      if (resp.answer && resp.answer.some(rr => rr.type === qtype)) {
        cache.set(domain, resp.answer, resp.ttl);
        return resp.answer;
      }
      // 若响应包含 NS 授权段和附加段中的粘接记录，更新下一跳服务器
      if (resp.authority && resp.additional && resp.additional.length > 0) {
        servers = resp.additional.filter(rr => rr.type === 'A' || rr.type === 'AAAA')
                                  .map(rr => rr.address);
        // currentName 保持不变，继续向更接近答案的服务器查询
        continue;
      }
      // 若响应为 CNAME，需跟随别名并从根重新开始迭代
      const cname = resp.answer.find(rr => rr.type === 'CNAME');
      if (cname) {
        currentName = cname.target;
        servers = rootHints;
        continue;
      }
      // 无答案且无引用，代表 NXDOMAIN 或无数据
      return [];
    }
  }

Node.js 内置模块对比：

  const dns = require('dns');
  // dns.lookup 调用系统解析器（通常为递归解析），使用 libuv 线程池，返回一个地址
  dns.lookup('example.com', (err, address) => console.log('系统递归:', address));
  // dns.resolve 直接向配置的 DNS 服务器发送请求，可观察递归解析器返回的最终记录
  dns.resolve('example.com', 'A', (err, records) => console.log('直接 DNS 记录:', records));

注意区分：dns.lookup 不经过 Node.js 内部的 DNS 实现，而是委托给操作系统；dns.resolve 使用 Node.js 的 c-ares 库发起标准 DNS 查询，更接近底层行为。
```

### 4. 常见误区与进阶思考
常见误区：1. 误以为根服务器会直接解析任何域名并返回最终 IP。实际上根服务器只返回顶级域（如 com.）的 NS 授权和粘接记录，不存储具体域名记录；最终答案由权威服务器提供。2. 混淆递归查询与迭代查询的参与者：认为客户端在标准解析中会与根、TLD、权威服务器直接通信。实际上客户端只向递归解析器发一次递归查询，后续迭代全部由递归解析器完成；只有使用 dig +trace 等工具手动模拟时才从客户端发起迭代。

进阶思考：假设全球所有根服务器同时不可用且递归解析器缓存已清空，新的域名解析请求是否会全部失败？请从根提示、缓存 TTL、转发器、私有根、以及浏览器/操作系统 DNS 缓存等层面分析哪些请求仍可能成功，并说明根服务器在什么情况下可以被绕过。
