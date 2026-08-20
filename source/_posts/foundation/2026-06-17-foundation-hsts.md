---
title: "每日基础技术总结 · 2026-06-17 · HSTS 与超域预加载及证书错误覆盖"
date: 2026-06-17 08:00:00
categories: [技术分享]
tags: ["技术分享", "安全基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-06-17 · HSTS 与超域预加载及证书错误覆盖

## 📚 今日主题

> **HSTS 与超域预加载及证书错误覆盖**（安全基础）

### 1. 核心概念速览
HSTS（HTTP Strict Transport Security）是一种由源服务器通过响应头 `Strict-Transport-Security` 告知用户代理的强制 HTTPS 策略。其本质是将“某域名必须使用 HTTPS 访问”这一安全约定从“每次请求时由服务器重定向/客户端主动升级”升级为“客户端在收到一次策略后，在有效期（max-age）内对该域名的所有请求一律在发送前强制改写为 HTTPS，且忽略任何证书错误”。它解决的核心问题是：首次 HTTP 请求或重定向过程中存在的中间人攻击（SSL Stripping）窗口，以及用户手动输入域名时默认走 HTTP 的风险。HSTS 属于 Web 安全体系中“传输层安全策略”的组成部分，位于 TLS 之上、应用层之下，与 CORS、CSP 等同属于浏览器强制执行的客户端安全机制。专业工程师必须掌握它，因为它直接影响站点可用性、证书迁移、子域覆盖与前端资源的加载行为，是理解 HTTPS 部署完整性的必要环节。

### 2. 底层原理剖析
HSTS 的底层机制分为三个层面：
1. 策略存储与生效域：浏览器首次通过 HTTPS 收到 `Strict-Transport-Security` 响应头时，解析其中 `max-age`（秒）、`includeSubDomains`、`preload` 三个指令，并将该策略以 {host, expiry, includeSubDomains} 形式存入内置的 HSTS 存储（通常为进程内持久化存储）。此后，在有效期内，用户代理内部会对该 host 及（若声明 includeSubDomains）其所有子域自动执行 URL 重写：凡是以 `http://` 形式发起的请求，在发出前直接改写为 `https://`，同时禁止用户绕过证书警告（即不允许点击“继续前往”）。
2. 超域预加载（HSTS Preload）：预加载列表是 Chrome 等浏览器内置的硬编码域名列表，由 hstspreload.org 审核后合入浏览器源码。其机制是：站点在响应头中携带 `preload` 指令并满足“max-age 至少 31536000、声明 includeSubDomains、且存在有效证书”等条件后，被提交至预加载列表。浏览器在编译或启动时加载该列表，因此对列表中的域名，即使从未访问过，也直接应用 HSTS 策略，彻底消除了“首次访问”窗口。这比普通 HSTS 更强，因为普通 HSTS 首次响应仍需通过 HTTPS 发出，若攻击者在该首次请求前拦截则仍可降级。
3. 证书错误覆盖：HSTS 策略生效后，浏览器对相关域名的 TLS 证书校验失败（如过期、自签、域名不匹配、链不完整）时，直接终止连接并显示硬性错误页，不提供“忽略警告”的绕过选项。其本质是 HSTS 将“必须使用可信证书”从浏览器默认的可配置行为提升为不可覆盖的强制约束。
伪代码表示：
```
function beforeRequest(url, hstsStore):
    if hstsStore.matches(url.host, includeSubDomains) or url.host in preloadList:
        if url.scheme == 'http':
            url.scheme = 'https'
    return url

function onCertificateError(host, error):
    if hstsStore.hasHost(host) or host in preloadList:
        return BLOCK_WITH_NO_OVERRIDE
    else:
        return SHOW_WARNING_WITH_OVERRIDE
```
与前端工程师已有概念的对比：HSTS 类似 `Cache-Control` 的客户端强制缓存——`Cache-Control: max-age=3600` 让浏览器在有效期内不重新请求，HSTS 让浏览器在有效期内强制 HTTPS；`includeSubDomains` 类似于 Service Worker 中 `clients.claim()` 对作用域的控制范围；`preload` 则类似于在浏览器安装时即注册的全局 Service Worker（不受站点首次访问限制）。但更本质的对比是：HSTS 不是“建议”而是“命令”，它把安全决策从服务器端重定向（`301` 跳转）转移到客户端 URL 合成阶段，从而消除了重定向链路中的明文暴露窗口。

### 3. 基础代码与实战验证
由于 HSTS 是协议层机制，无法用纯前端 JavaScript 直接触发浏览器内部存储，但可以通过标准 Web API 验证其部分行为。以下为使用 Node.js 原生 `http`/`https` 模块模拟 HSTS 策略处理逻辑，并展示证书错误覆盖的判定：

```javascript
// hsts-simulate.js
// 模拟浏览器 HSTS 存储与预加载列表
const hstsStore = new Map();
const preloadList = new Set(['example.com']);

// 解析并存储 HSTS 头
function parseAndStoreHSTS(host, headerValue) {
  if (!headerValue) return;
  const maxAgeMatch = /max-age=(\d+)/.exec(headerValue);
  const includeSubDomains = /includeSubDomains/.test(headerValue);
  const preload = /preload/.test(headerValue);
  if (maxAgeMatch && Number(maxAgeMatch[1]) > 0) {
    hstsStore.set(host, {
      expiresAt: Date.now() + Number(maxAgeMatch[1]) * 1000,
      includeSubDomains,
      preload
    });
  }
}

// 在请求发出前，改写 URL 协议
function upgradeToHttps(url) {
  const u = new URL(url);
  const host = u.hostname;
  const policy = hstsStore.get(host);
  const isPreloaded = preloadList.has(host);
  const isHostCovered = (policy && policy.expiresAt > Date.now()) || isPreloaded;
  const isSubdomainCovered = policy && policy.includeSubDomains &&
    host.endsWith('.' + [...hstsStore.keys()].find(k => host !== k && host.endsWith('.' + k)));
  if ((isHostCovered || isSubdomainCovered) && u.protocol === 'http:') {
    u.protocol = 'https:';  // 关键：在发送前直接改写，避免明文请求
  }
  return u.toString();
}

// 模拟证书错误处理：HSTS 生效时禁止绕过
function onCertificateError(host) {
  const policy = hstsStore.get(host);
  const isPreloaded = preloadList.has(host);
  if ((policy && policy.expiresAt > Date.now()) || isPreloaded) {
    return 'BLOCK: 证书错误，不可覆盖';  // HSTS 强制终止
  }
  return 'WARN: 可点击继续';  // 非 HSTS 时可忽略
}

// 验证流程
parseAndStoreHSTS('secure.example.com', 'max-age=31536000; includeSubDomains; preload');
console.log(upgradeToHttps('http://secure.example.com/path')); // https://secure.example.com/path
console.log(upgradeToHttps('http://sub.secure.example.com/x')); // 子域也升级
console.log(upgradeToHttps('http://other.com/y')); // 未覆盖，保持不变
console.log(onCertificateError('secure.example.com')); // BLOCK: 证书错误，不可覆盖
console.log(onCertificateError('other.com')); // WARN: 可点击继续
```

关键点注释：
- 第 5 行：HSTS 存储以 host 为键，策略包含过期时间和子域覆盖标志。
- 第 18 行：`upgradeToHttps` 模拟浏览器在 URL 解析后、实际网络请求前对协议进行改写，这是 HSTS 的核心动作。
- 第 27 行：判断子域覆盖时，需从存储中寻找最接近父域的策略，实际浏览器实现更复杂（按域名后缀匹配）。
- 第 35 行：证书错误处理中，HSTS 策略直接决定是否允许绕过警告，这是对用户强制安全的体现。

### 4. 常见误区与进阶思考
常见误区：
1. 认为只要部署了 HTTPS 并返回 HSTS 头就绝对安全。实际上，若未加入预加载列表且首次请求仍可通过 HTTP 发起，攻击者可在首次响应前拦截并剥离 HSTS 头（即 SSL Stripping）。即使响应头由 HTTPS 返回，攻击者也可能通过中间人伪造证书（若用户已信任错误证书）或利用代理缓存破坏。正确做法是：先通过 HTTPS 部署，再逐步加强 max-age，最终提交预加载列表，并在 HSTS 头生效前确保所有子域都已支持 HTTPS。
2. 混淆“证书错误覆盖”与“HSTS 头本身”的信任关系。HSTS 策略必须由可信的 HTTPS 响应携带才会被存储；如果用户首次就遇到证书错误并选择忽略，浏览器不会存储该响应中的 HSTS 头。因此，一旦用户手动绕过了一次证书错误，后续该域名的 HSTS 策略就不会被设置，攻击者可以继续降级攻击。

进阶思考题：
假设一个域名 `example.com` 已在 Chrome 预加载列表中，且列表要求 `includeSubDomains`。现在你希望在 `b.example.com` 上仅使用 HTTP 提供一张公开图片，浏览器会如何处理该请求？请从 URL 改写顺序、子域策略匹配优先级、以及能否通过 `Strict-Transport-Security` 响应头覆盖预加载策略三个角度分析，并说明这是否意味着预加载列表中的域名永远无法为子域提供明文服务？为什么？
