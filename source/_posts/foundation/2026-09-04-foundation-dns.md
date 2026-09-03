---
title: "每日基础技术总结 · 2026-09-04 · DNS 解析流程"
date: 2026-09-04 07:01:36
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-04 · DNS 解析流程

## 📚 今日主题

> **DNS 解析流程**（前端底层与计算机基础）

### 1. 核心概念速览
DNS（Domain Name System）是互联网的分布式层次命名系统，本质是将人类可读的域名映射为机器可寻址的IP地址（及其他资源记录）的全球数据库。它解决的是“符号寻址”与“数字寻址”之间的转换问题，是应用层（HTTP、SMTP等）与网络层（IP）之间的关键粘合剂。DNS并非单一服务器，而是一个由根域名服务器、顶级域服务器、权威服务器和本地递归解析器组成的多层分布式树状结构，通过缓存与TTL机制平衡一致性与性能。专业工程师必须掌握其解析流程，因为它是排查网络故障、优化首字节延迟、理解CDN调度、实施零信任安全策略（如DoH/DoT）以及设计高可用架构的底层基础。

### 2. 底层原理剖析
DNS解析流程遵循“递归查询 + 迭代查询”的混合模式。客户端（stub resolver）通常只与本地递归解析器（LDNS，如8.8.8.8或运营商DNS）通信，LDNS代表客户端完成完整的遍历。

详细步骤（以解析 www.example.com 为例）：
1. 客户端检查本地DNS缓存（浏览器缓存 → OS缓存 → hosts文件），若命中直接返回。
2. 未命中则向LDNS发送递归查询请求（默认UDP 53端口，超过512字节转TCP）。
3. LDNS首先检查自身缓存，若命中返回；否则向根服务器发送迭代查询。
4. 根服务器不保存具体记录，但返回 .com 顶级域服务器的NS记录及对应IP（若LDNS已有根提示，可跳过直接查询顶级域服务器）。
5. LDNS向 .com 顶级域服务器查询 example.com，顶级域返回 example.com 的权威服务器NS记录及IP。
6. LDNS向 example.com 的权威服务器查询 www 的A/AAAA记录，权威服务器返回最终的IP。
7. LDNS将结果缓存（TTL由权威服务器指定），同时返回给客户端并写入客户端缓存。

底层机制要点：
- 每条DNS记录都有类型（A/AAAA/NS/CNAME/MX等），解析过程中逐级委派（delegation）实现非集中式管理。
- CNAME记录会触发额外查询链：若www是CNAME指向cdn.example.net，LDNS需重复步骤获取cdn.example.net的A记录。
- 根服务器仅提供顶级域委派，不参与业务域名解析，因此流量极小但至关重要。
- 递归与迭代的区别：递归是“代为完成并返回最终结果”，迭代是“返回下一步的指引”。

与前端已有概念的对比：
- 类似“接口调用链”：前端通过SDK调用一个接口，SDK内部可能经历多次重定向（302）最终返回数据，如同LDNS遍历各级服务器。
- 类似“对象原型链”：JS原型链沿__proto__逐级向上查找，直至null；DNS沿域名层级从根开始向下委派，都是“层级化查找”模式。
- 但本质不同：JS原型链是单一对象的内存结构，查找确定性且无网络开销；DNS是多层分布式网络交互，依赖缓存、超时和容错机制。

### 3. 基础代码与实战验证
由于DNS解析是系统级网络行为，无法用纯前端代码直接实现底层协议，但可以通过Node.js的dns模块验证核心流程（该模块封装了系统c-ares库）。

```javascript
const dns = require('dns');

// 1. 使用系统解析器执行递归查询（走/etc/resolv.conf配置的LDNS）
dns.resolve('www.baidu.com', 'A', (err, addresses) => {
  if (err) console.error('解析失败', err);
  else console.log('A记录:', addresses);
});

// 2. 获取CNAME链，观察解析过程中的别名重定向
dns.resolveCname('www.github.com', (err, records) => {
  if (err) console.error('无CNAME', err);
  else console.log('CNAME链:', records);
});

// 3. 直接查询权威服务器，绕开缓存，模拟迭代查询中的最终步骤
// 先手动获取github.com的权威NS记录，再向该权威服务器请求A记录
const ns = require('dns');
ns.resolveNs('github.com', (err, nsList) => {
  if (err) throw err;
  console.log('权威NS:', nsList);
  // 使用指定服务器解析（模拟LDNS向权威查询）
  const resolver = new dns.Resolver();
  resolver.setServers([nsList[0]]);  // 指向权威服务器
  resolver.resolve('www.github.com', 'A', (e, ip) => {
    console.log('权威服务器返回IP:', ip);
  });
});
```

关键注释：
- `dns.resolve` 使用系统配置的LDNS发起递归查询，底层走UDP/TCP 53端口，结果会被系统缓存（TTL内）。
- `dns.resolveCname` 仅返回CNAME记录，不展开后续A记录查询，可用于观察CDN别名链。
- `dns.Resolver` 可自定义DNS服务器，通过`setServers`指向权威服务器，从而绕过本地缓存，直接验证权威解析结果，模拟完整解析流程中的最后一跳。

### 4. 常见误区与进阶思考
常见误区：
1. 认为DNS解析只是一次“域名→IP”的简单映射。实际可能涉及多次递归（CNAME链）、DNS负载均衡（同一域名返回多个IP）、GeoDNS根据源地址返回不同结果，甚至HTTP层重定向。前端工程师常误以为一次解析拿到IP就结束，忽略CNAME对性能的影响（每多一次CNAME解析都会增加一次RTT）。
2. 忽略缓存层次与TTL的相互作用。浏览器、操作系统、LDNS、权威服务器各级缓存策略不同，修改DNS记录后生效时间取决于最坏路径上的TTL，而非仅权威配置的TTL。专业工程师在切换CDN或迁移服务器时，常因未预估缓存而出现部分用户访问旧IP的问题。

深度思考题：
假设客户端本地DNS缓存中已经存在 `www.example.com` 的A记录，但该域名同时配置了CNAME指向另一域名，且CNAME记录刚被修改。客户端在TTL未过期前发起解析，是否会感知到CNAME变化？为什么？请从DNS缓存粒度（是否区分CNAME记录与A记录）和迭代查询触发条件的角度分析。
