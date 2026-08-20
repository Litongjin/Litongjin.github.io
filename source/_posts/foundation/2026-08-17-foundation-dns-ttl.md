---
title: "每日基础技术总结 · 2026-08-17 · DNS 的 TTL 与缓存一致性"
date: 2026-08-17 08:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-08-17 · DNS 的 TTL 与缓存一致性

## 📚 今日主题

> **DNS 的 TTL 与缓存一致性**（网络基础）

### 1. 核心概念速览
DNS TTL（Time To Live）是资源记录（RR）中携带的生存时间，单位为秒，它定义了一条 DNS 应答可以被解析器（Resolver）或操作系统缓存的最长时长。其本质是分布式缓存系统的一致性时间窗口：TTL 不是绝对失效时间，而是服务端允许客户端对旧数据持续信任的上限。它解决的核心问题是：在无需让每个客户端每次查询都穿透到权威服务器的情况下，通过可控的过期策略平衡查询延迟、权威服务器负载与数据变更传播速度。DNS 的 TTL 位于整个互联网域名解析链路中的缓存层级之间，包括浏览器缓存、OS 缓存、递归解析器缓存、权威服务器自身。专业工程师必须掌握它，因为它是构建高可用服务（如切流、故障转移、CDN 调度、灰度发布）时，变更生效延迟的最底层决定因素；同时它也是理解 HTTP 缓存（Cache-Control max-age）思想在更底层协议中的对应物，但机制与粒度截然不同。

### 2. 底层原理剖析
DNS 解析是一个多级缓存协作的读取模型。客户端发起解析请求时，先查浏览器缓存和 OS 缓存，未命中则发送到递归解析器（如 8.8.8.8），递归解析器沿着根、顶级域、权威服务器的链逐级查询，最终获得权威应答。权威应答中每条记录附带 TTL，递归解析器缓存该记录，并在 TTL 内对后续查询直接返回缓存结果。TTL 的递减机制：递归解析器在缓存期间，如果收到一个 TTL 为 3600 的记录，它会在自己的缓存中保留该记录，并在这 3600 秒内对客户端响应时返回一个递减后的 TTL（即剩余时间）。这导致一个关键性质：TTL 在每级缓存中不是简单的重复缓存，而是不断被削減，因此离客户端越近的缓存，其 TTL 值越小。但客户端（浏览器/OS）通常只看第一次拿到时的 TTL，然后在本地倒计时，所以整体一致性的上限取决于初始 TTL。

底层逻辑可以用以下伪代码精确描述：

```
// 递归解析器查询逻辑
function resolve(name) {
  entry = cache.lookup(name);
  if (entry && entry.expireAt > now()) {
    entry.ttl_remaining = entry.expireAt - now();
    return { ip: entry.ip, ttl: entry.ttl_remaining };
  }
  // 未命中或过期，向权威服务器发起迭代查询
  response = queryAuthoritative(name);
  record = response.answer[0];
  cache.store(name, record.ip, now() + record.ttl);
  return { ip: record.ip, ttl: record.ttl };
}
```

注意这里 TTL 是一个“倒计时剩余时间”，而不是绝对时间戳。每个缓存节点各自计算自己的过期时刻，因此整个链路的实际生效时间是初始 TTL 与各级缓存延迟之和。如果权威记录 TTL=300，但某级递归缓存已缓存 299 秒，那么客户端拿到后仍可能再缓存 300 秒，导致最长可达初始 TTL + 各级剩余时间。

与前端已有概念对比：DNS TTL 类似于 HTTP 的 `Cache-Control: max-age`，但有几个关键区别：
1. 粒度：DNS 缓存对象是资源记录（A/AAAA/CNAME 等），HTTP 缓存对象是响应体。
2. 验证机制：HTTP 有 `ETag/Last-Modified` 进行条件请求（revalidation），DNS 没有标准的基于值的验证，只能等 TTL 过期后重新查询。
3. 层级：DNS 的每级缓存（浏览器→OS→递归器→权威）都会独立应用 TTL，而 HTTP 的代理缓存与浏览器缓存层级类似但控制头可以覆盖（如 `no-cache`、`s-maxage`）。
4. 时间粒度：DNS TTL 通常为 30~86400 秒，HTTP max-age 可以到秒以下甚至 0。
5. 一致性模型：DNS 是最终一致性，且没有原子切换；HTTP 缓存则通过请求头字段（如 `Cache-Control: no-cache`）让客户端强制每次校验。

本质上来讲，TTL 是缓存一致性中的“租约”机制：权威服务器将“我可以信任这个数据一段时间”的权利租给客户端，过期后必须重新续租。没有 TTL，DNS 要么每次全量穿透（性能灾难），要么永久缓存（变更永远不生效）。TTL 就是性能与一致性之间的旋钮。

### 3. 基础代码与实战验证
由于 DNS TTL 的行为本质上发生在系统库和网络协议栈中，这里给出两段纯基础代码来验证缓存效果和 TTL 递减。

1) 用 Python 标准库验证 DNS 缓存的 TTL 递减：

```python
import dns.resolver  # 需要 dnspython，但核心逻辑不依赖框架

# 第一次查询，拿到权威返回的 TTL（通常为 300/3600 等）
answer = dns.resolver.resolve('example.com', 'A')
first_ttl = answer.rrset.ttl
print(f'第一次拿到 TTL: {first_ttl}')

import time
time.sleep(2)

# 第二次查询，如果使用同一个 resolver 对象，dnspython 默认不缓存，所以会再次走网络，
# 但这里演示的是：我们自己手动模拟一个缓存层
cached_until = time.time() + first_ttl
for _ in range(3):
    remaining = cached_until - time.time()
    print(f'当前剩余 TTL: {remaining:.1f}s')
    time.sleep(1)
# 实际运行中，第二次查询如果从系统缓存获得，则 TTL 会变小；
# 如果系统不支持查看 TTL，可用 `dig` 命令观察：dig @8.8.8.8 example.com +noall +answer 两次查询的 TTL 差异。
```

2) 用 `dig` 命令验证递归解析器返回的 TTL 递减（伪代码/步骤）：

```
步骤1：清空本地缓存（Linux: sudo systemd-resolve --flush-caches; macOS: sudo killall -HUP mDNSResponder）
步骤2：dig @1.1.1.1 example.com A +noall +answer
步骤3：等待 10 秒，再次执行相同命令
步骤4：对比两次输出的 TTL 数值，第二次的 TTL 通常比第一次小（或相同，取决于服务器返回策略），但不会大于第一次。

说明：
- 第一次返回的 TTL 是权威记录的真实 TTL。
- 第二次返回的 TTL 是递归服务器中该记录剩余的生存时间，证明 TTL 在缓存层被递减。
- 如果递归服务器配置了“缓存未过期时返回原 TTL”的选项（如 BIND 的 max-cache-ttl 与 min-cache-ttl），则可能不递减，但标准实现会递减。
```

对于前端工程师，可以类比 HTTP 缓存验证：

```javascript
// 前端模拟 DNS TTL 的缓存逻辑
const dnsCache = new Map();
function resolveWithTTL(name, getAnswer) {
  const now = Date.now();
  const entry = dnsCache.get(name);
  if (entry && entry.expireAt > now) {
    return {
      ip: entry.ip,
      ttl: Math.floor((entry.expireAt - now) / 1000)
    };
  }
  const answer = getAnswer(name); // 假设返回 {ip, ttl}
  dnsCache.set(name, { ip: answer.ip, expireAt: now + answer.ttl * 1000 });
  return answer;
}
```

这行代码 `expireAt: now + answer.ttl * 1000` 体现了 TTL 的本质：将服务端给定的秒数转换为本地绝对过期时间，后续所有查询只判断过期时间，不重新询问服务端。这就是缓存一致性的全部秘密。

### 4. 常见误区与进阶思考
误区1：认为 TTL 是绝对时间戳，过了 TTL 后所有缓存同时失效。实际上每个缓存节点独立计算自己的过期时刻，一个记录在不同层级可能已经存在不同的时间，所以实际生效时间会大于初始 TTL。例如：权威设置 TTL=60，但某个递归缓存已经缓存了 30 秒，客户端再缓存 60 秒，最坏情况下 90 秒后才看到变更。因此做 DNS 变更（如切 IP）时，必须等待“所有递归服务器剩余 TTL + 客户端本地 TTL”的全局最大时间，而不是简单的 TTL 时间。

误区2：以为 TTL 越小越好，就能立即生效。实际上 TTL 过小会导致权威服务器负载急剧上升，因为递归服务器在 TTL 到期后会重新查询。同时，客户端（浏览器/OS）可能存在最小 TTL 限制（如 Windows 默认将 TTL 低于 5 秒的视为 5 秒），导致你设置的 1 秒 TTL 不会按预期生效。此外，某些递归服务器（如运营商）会忽略权威返回的较小 TTL 而强制使用自己的最小值。

进阶思考题：假设你的域名记录在权威服务器上从 IP A 变更为 IP B，且权威 TTL 设置为 300 秒。同时你有一个递归解析器集群，其中 30% 的服务器在变更前 120 秒已缓存了该记录，另外 70% 在变更前 10 秒才缓存。请问：在权威变更后的什么时间点，可以保证所有客户端（包括浏览器和 OS 缓存）一定解析到新 IP？请给出最坏情况的时间计算，并说明为什么不能仅仅依赖权威 TTL。
