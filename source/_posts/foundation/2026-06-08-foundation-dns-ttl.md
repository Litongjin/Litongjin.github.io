---
title: "每日基础技术总结 · 2026-06-08 · DNS 缓存 TTL 与缓存污染"
date: 2026-06-08 08:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-06-08 · DNS 缓存 TTL 与缓存污染

## 📚 今日主题

> **DNS 缓存 TTL 与缓存污染**（网络基础）

### 1. 核心概念速览
DNS 缓存 TTL（Time to Live）是权威 DNS 服务器在响应资源记录时携带的存活时间值，本质是分布式命名系统下缓存一致性与查询性能之间的显式权衡机制。它解决递归解析器在多大时间内可安全复用已查询结果的问题，避免每次域名解析都触发完整的根→TLD→权威链路迭代查询。TTL 由权威服务器按记录属性与业务需求设定，递归解析器与客户端（操作系统、浏览器）各自独立执行超时淘汰。缓存污染则是指恶意或异常来源的数据被写入缓存并持续到 TTL 过期，导致终端解析到伪造的 IP，是 DNS 安全模型中最核心的破坏场景。该知识点位于网络基础与应用层中间，是理解解析链、DDoS 防御、CDN 流量调度以及劫持攻击的基石；专业工程师必须掌握它才能在配置域名变更、设计高可用系统及评估安全风险时做出正确判断。

### 2. 底层原理剖析
底层机制：客户端（如浏览器）发起域名解析时，先查本地缓存（浏览器缓存/OS 缓存），未命中则发给递归解析器。递归解析器维护一张 key=域名/类型、value=记录+到期时间 的表。每次命中时检查 expiry = insert_time + TTL，若未过期直接返回；否则向权威服务器发起迭代查询，获得新记录后重新计算 expiry。TTL 不是由递归解析器或客户端设定，而是权威服务器在下发记录时写入的。因此，控制 TTL 的是数据发布方，而非消费方。

与前端 HTTP 缓存的对比：HTTP 强缓存通过 Cache-Control 的 max-age 指定，语义与 TTL 类似，但 HTTP 还提供协商缓存（ETag/Last-Modified）来验证资源是否变化，而 DNS 没有独立的验证机制，唯一的一致性手段就是等 TTL 过期。另外，HTTP 缓存通常作用于单个资源，而 DNS 缓存作用于全局命名空间，且 DNS 查询链上每一级（客户端、递归器）都可能有自己的缓存策略，形成多层失效延迟。

伪代码描述递归解析器缓存逻辑：

struct DnsRecord { ip; expireAt; }
Map cache;

function resolve(domain) {
    if cache.contains(domain) and cache[domain].expireAt > now():
        return cache[domain].ip;
    response = upstreamQuery(domain);  // 迭代/递归查询权威
    ttl = response.ttl;
    cache[domain] = { ip: response.ip, expireAt: now() + ttl };
    return response.ip;
}

缓存污染的发生：攻击者通过伪造响应（如提前猜测查询 ID、利用 UDP 无状态性）将恶意记录注入递归解析器，或权威服务器被攻破下发错误记录。由于缓存器无法验证记录真实性（DNSSEC 缺失），污染记录会一直存活到 TTL 归零，期间所有使用该递归器的客户端都会被劫持。TTL 越短，污染窗口越小，但缩短 TTL 会增加权威服务器查询压力，这是本质权衡。

### 3. 基础代码与实战验证
```text
以下用 Python 模拟一个带 TTL 的 DNS 缓存及污染效果。代码不依赖外部库，直接演示缓存过期与污染注入。

import time

class DnsCache:
    def __init__(self):
        self.table = {}  # 域名 -> (ip, expire_at)

    def lookup(self, domain):
        now = time.time()
        entry = self.table.get(domain)
        if entry and entry[1] > now:
            return entry[0]
        return None

    def update(self, domain, ip, ttl):
        self.table[domain] = (ip, time.time() + ttl)

# 模拟权威响应
cache = DnsCache()
cache.update('example.com', '1.2.3.4', ttl=5)
print(cache.lookup('example.com'))  # 输出 1.2.3.4

# 模拟缓存污染：攻击者注入错误 IP，TTL 设为 10
cache.update('example.com', '6.6.6.6', ttl=10)
print(cache.lookup('example.com'))  # 输出 6.6.6.6（污染生效）

# 等待 5 秒后原始记录已过期，但污染记录仍在（因为污染 TTL=10）
time.sleep(6)
print(cache.lookup('example.com'))  # 仍然输出 6.6.6.6，证明污染可持续到 TTL 结束

关键注释：
- update 中 time.time() + ttl 决定记录过期时刻，所有缓存命中都必须判断当前时间是否早于该值。
- 污染者只要能在缓存未过期前注入，就能将错误记录保留至自己设定的 TTL，演示了 TTL 是唯一失效机制。
- 实际递归器中还需要验证查询 ID、源端口等，但核心逻辑就是这个表。
```

### 4. 常见误区与进阶思考
误区1：误认为 TTL 是缓存方（客户端或递归器）可以自由调整的。实际上 TTL 来自权威服务器响应，缓存方只能选择遵守或忽略（部分解析器会强制上限），一旦修改就破坏原始意图，可能导致域名变更延迟或提前失效。

误区2：把缓存污染等同于 DNS 劫持。DNS 劫持是中间设备（如运营商）强制改写响应，而缓存污染是攻击者将伪造记录写入递归器缓存；两者的攻击位置和持久性不同。污染记录在 TTL 内即使源已修复也不会自动纠正，需要刷新缓存或等过期。

思考题：为什么“缩短 TTL 可以降低缓存污染影响”这句话是片面的？
答案：缩短 TTL 确实缩短了单次污染的有效期，但如果攻击者控制权威服务器（或能持续伪造响应），他可以每次都以短 TTL 重新注入污染，使得影响持续存在，同时增加了递归器的查询频率，消耗上游资源。因此，TTL 的缩短只是缩小了单次攻击窗口，并不能消除污染风险，反而可能放大 DDoS 影响。真正的防护是 DNSSEC 验证和权威服务器安全。
