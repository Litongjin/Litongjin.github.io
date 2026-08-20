---
title: "每日基础技术总结 · 2026-06-08 · DNS 递归与迭代解析"
date: 2026-06-08 08:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-06-08 · DNS 递归与迭代解析

## 📚 今日主题

> **DNS 递归与迭代解析**（网络基础）

### 1. 核心概念速览
DNS 递归与迭代解析是域名解析流程中客户端与各级 DNS 服务器之间的两种交互模式。递归解析指客户端将完整解析任务委托给某个 DNS 服务器，由该服务器负责逐级查询并最终返回最终结果（或错误）；迭代解析则指 DNS 服务器在无法直接回答时，向客户端返回下一级可查询的服务器地址，由客户端或上层递归器继续发起查询。其本质是分布式数据库的查询策略：递归是'代查'，迭代是'指引'。该机制解决的是将人类可读域名映射为机器可读 IP 地址的全局性问题，是互联网可用性的基石。它在网络分层中位于应用层（DNS 协议），但依赖传输层 UDP/TCP 和底层 IP 路由。专业工程师必须掌握它，因为这是诊断网络故障、设计高可用服务、理解 CDN 调度和 HTTP/3 连接建立（如 ECH）的基础；前端工程师尤其需要理解浏览器解析域名到发起请求的完整链路，避免将网络延迟误判为应用性能问题。

### 2. 底层原理剖析
底层机制如下：客户端（如浏览器）首先检查本地缓存和 hosts 文件，若无则向配置的本地递归解析器（通常由 ISP 或公共 DNS 提供）发起递归查询。递归解析器接受该任务后，自身扮演'客户端'角色，对根服务器进行迭代查询：1) 请求根服务器，根返回顶级域（如 .com）的权威服务器地址；2) 递归器再请求该 TLD 服务器，获取二级域（如 example.com）的权威服务器地址；3) 递归器请求该权威服务器，获得最终 A/AAAA 记录。这整个过程对原始客户端是透明的，客户端只发起一次查询并等待最终结果。若递归器配置为迭代模式，则每次响应都只是'下一步该问谁'的指引。本质区别在于状态管理：递归服务器维护完整查询状态并负责失败重试；迭代查询则无状态，每次响应独立。伪代码：

function recursiveResolve(domain, client):
    cache = checkCache(domain)
    if cache: return cache
    result = iterativeResolve(domain)
    storeCache(domain, result)
    return result

function iterativeResolve(domain):
    server = rootServer
    while true:
        response = query(server, domain)
        if response is final: return response
        server = response.nextServer

与前端概念的对比：这类似于 Java 的接口与 TypeScript 的接口的区别——递归和迭代看似都是'查询'，但语义截然不同。Java 接口是运行时强制契约（相当于递归：发起方必须等待完整结果），而 TS 接口是编译期结构约束（相当于迭代：只提供下一步的指引，不保证最终结果）。更贴切的对比是 HTTP 重定向与代理：递归类似反向代理（客户端无感知，代理完成所有子请求），迭代类似 3xx 重定向（客户端被指引到另一个地址，需自行发起新请求）。前端工程师熟悉的事件循环中，递归对应同步阻塞等待，迭代对应异步回调链——但 DNS 的迭代查询本质上是同步的，由发起方（递归器）顺序执行。

### 3. 基础代码与实战验证
```text
由于 DNS 是协议而非库，这里用精确的伪代码模拟浏览器端和递归解析器的交互：

# 客户端代码（如浏览器）
function resolveDomain(domain) {
  // 1. 检查本地 DNS 缓存
  if (localCache.has(domain)) return localCache.get(domain);
  
  // 2. 向配置的递归解析器发起递归查询（通常通过 UDP 53 端口）
  // 递归标志 RD=1，表示要求服务器代查
  const response = dnsQuery({ name: domain, type: 'A', recursiveDesired: true });
  
  // 3. 响应中携带最终 IP 或错误码，客户端无需关心中间过程
  return response.answer;
}

# 递归解析器内部（简化）
function recursiveResolve(domain) {
  // 对根服务器发起迭代查询，RD 标志通常为 0（因为递归器自己承担迭代）
  let server = getRootServer();
  while (true) {
    // 向当前 server 发送查询，期望返回 NS 或 A 记录
    const resp = dnsQuery({ server, name: domain, recursiveDesired: false });
    if (resp.answer) {
      // 拿到最终 IP，写入缓存并返回给客户端
      cache.set(domain, resp.answer);
      return resp.answer;
    }
    // 否则响应中给出下一级权威服务器的 IP，更新 server 继续循环
    server = resp.nextServer;
  }
}

# 用真实命令验证（终端）：
# dig +trace example.com  # 显示从根开始的完整迭代路径
# dig example.com @8.8.8.8  # 观察递归解析器返回的最终结果，加 +norecurse 可模拟迭代查询

注意：实际递归器会缓存根和 TLD 信息，并不会每次从根开始；但原理相同。
```

### 4. 常见误区与进阶思考
误区一：认为迭代查询是客户端直接与根服务器交互。实际上普通客户端的操作系统或浏览器只发起递归查询，迭代过程由递归解析器完成。前端工程师常误以为每次域名解析都经过根服务器，导致性能焦虑；实际递归器有大量缓存，且操作系统/浏览器也有缓存。

误区二：混淆 DNS 缓存与 HTTP 缓存。DNS 缓存由 TTL 控制，但本地递归器可能忽略 TTL 或刷新策略（如 Chrome 的 80 秒缓存），导致解析结果更新延迟。这与 HTTP 的 Cache-Control 语义完全不同，排障时需区分。

思考题：如果一台权威 DNS 服务器对同一域名返回了不同的 A 记录（例如基于地理位置），但你的本地递归解析器启用了 EDNS Client Subnet（ECS）功能，那么迭代查询过程中哪一层会根据客户端 IP 进行决策？如果递归器不转发 ECS，最终解析结果可能对客户端是非最优的——请解释为什么，并指出在递归与迭代的哪个环节丢失了客户端 IP 信息。
