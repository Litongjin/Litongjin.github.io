---
title: "每日基础技术总结 · 2026-10-28 · Service Worker 拦截与缓存策略：Cache-first 与 Network-first 的应用场景"
date: 2026-10-28 07:03:24
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-28 · Service Worker 拦截与缓存策略：Cache-first 与 Network-first 的应用场景

## 📚 今日主题

> **Service Worker 拦截与缓存策略：Cache-first 与 Network-first 的应用场景**（前端底层与计算机基础）

### 1. 核心概念速览
Service Worker 是注册在 ServiceWorkerGlobalScope（独立 worker 线程、无 DOM、事件驱动、可被浏览器随时终止）中的脚本，它在受控页面的网络请求链路上插入一个可编程的代理层：页面每发起一次 fetch/navigation，浏览器会先检查该请求的 URL 是否落在 SW 的 scope 内、发起请求的 client 是否受控，若是则向 SW 线程派发 FetchEvent，由 SW 在事件回调的同步阶段调用 event.respondWith(promise) 接管响应，否则回落到默认网络栈。其本质是把「请求→响应」的映射从浏览器的声明式 HTTP 缓存（Cache-Control/ETag/Vary，由网络栈自动执行、不可编程）下沉为应用级命令式控制，实现离线可用、弱网降级、细粒度资源策略与版本化发布。CacheStorage 是一份 Request → Response 的持久化私有 Map，与 HTTP 缓存相互独立（但 fetch() 发起的真实请求默认仍会命中 HTTP 缓存）。

两种策略是同一个缓存一致性模型的两端：Cache-first（先 match 缓存，miss 再走网络并回填）选择可用性与延迟，放弃新鲜度——适用于内容不可变的资源（content-hash 命名的 JS/CSS、字体、图片）；Network-first（先走网络，失败/超时才回落缓存）选择新鲜度，放弃延迟与弱网可用性——适用于变更频繁、必须反映真值源的资源（HTML 文档入口、API GET、用户态数据）。

在整个体系中的位置：HTTP 缓存（单机、协议级）→ SW CacheStorage（单机、应用级、私有缓存、键可自定义）→ CDN/反向代理（共享缓存、边缘侧）。SW 相当于把 CDN 的边缘策略下沉到客户端，并且拥有 CDN 不具备的维度（设备状态、用户身份、可编程降级）。专业工程师必须掌握的原因：它是 Web 平台唯一能以代码形式控制请求-响应链路的 API，直接决定 PWA 的离线能力、TTFB/LCP 指标、发布一致性（新旧版本混用）以及安全边界（secure context、scope、opaque response 的处理）。

### 2. 底层原理剖析
一、线程模型与生命周期
- SW 不在主线程，无 window/DOM，只能通过 postMessage 与 client 通信；空闲超时会被终止，因此禁止依赖内存中的可变状态，状态必须落到 CacheStorage / IndexedDB。
- 生命周期：register → install（precache、失败则整套作废）→ installed(waiting) → activating → activated → redundant。脚本更新走字节级比对，浏览器对 SW 脚本做强制更新检查（updateViaCache 决定 HTTP 缓存是否参与）；waiting 中的新版本要等所有受控 client 关闭，或由 skipWaiting() + clients.claim() 强制接管（代价是新旧版本可能共存）。
- 控制权边界：首次注册的页面本次加载不受控；scope 决定可拦截的 URL 前缀；非 GET、跨域 opaque 响应、部分受限请求需要显式判断后放行。

二、拦截链路（关键语义）
页面发起 fetch(url)
  → 存在 active SW 且 url ∈ scope 且 client 受控？
     否 → 默认网络栈（HTTP 缓存 → 网络）
     是 → 跨线程派发 FetchEvent（request 被结构化克隆，body 为流，消费前必须 clone）
          → 同步阶段内调用了 respondWith(p)？
              否 → 默认网络栈
              是 → p resolve(Response) → 交给渲染进程，HTTP 缓存层被绕过（除非 Response 来自 fetch()）
                   p reject → 直接变成网络错误（NetworkError），不会自动回落

三、策略的本质：缓存一致性模型的选择
- Cache-first：读路径优先命中本地私有存储，写路径惰性回填。有效命中率 ≈ 资源内容变更频率的倒数；对不可变资源趋近 100%。一致性上是「弱一致 + 高可用 + 低延迟」。
- Network-first：以网络为真值源，缓存仅作降级副本。每次请求付出一次网络 RTT，弱网下会被网络延迟/挂起拖累（需自行用 AbortController 加超时）。一致性上是「强一致取向 + 牺牲延迟与离线可用性」。
- 折中：stale-while-revalidate（立即返回缓存、后台刷新并回填）、cache-then-network-with-timeout。

四、与前端已有概念的对照
- HTTP 缓存：声明式、协议级、由网络栈执行，键为 URL+Vary，受 Cache-Control/ETag 约束，无法插入分支逻辑；SW 是命令式、应用级，可自定义键（ignoreSearch/ignoreVary、按 request 对象整体匹配），可在缓存之外做降级、改写、合成响应。
- Node/Express 中间件：同为 request pipeline（fetch 事件相当于 handler，respondWith 相当于结束响应，没有 next()），但 SW 无服务端共享状态、可能被随时杀死、且运行在客户端而非服务端，缓存是私有的。
- CDN 边缘缓存：共享缓存、多用户复用、命中率高但粒度粗；SW 是私有缓存，可结合用户态做个性化与离线降级，命中率低但语义强。
- CacheStorage ≠ 内存缓存：它是持久化存储，受 quota 与「存储压力下整体驱逐」影响（可 navigator.storage.persist() 申请持久化），不会像 HTTP 缓存那样按 max-age 自动过期，过期与清理必须自己实现。

五、CacheStorage 语义细节
- 结构是 Map<Request, Response>，按插入顺序；match 默认只处理 GET/HEAD（HEAD 由 GET 命中），匹配时考虑 Vary，可用 ignoreSearch/ignoreVary 放宽。
- Response body 是 ReadableStream，只能消费一次：cache.put(req, res) 之后再 return res 会读空，必须先 res.clone()。
- cache.put 不接受 206/部分响应，且 res.ok 与 res.type 需自行校验，否则会把错误响应写入缓存。

### 3. 基础代码与实战验证
```text
// ============ 1) 页面侧注册（index.html 内的内联脚本） ============
if ('serviceWorker' in navigator) {
  // updateViaCache:'none' 让 SW 脚本自身绕过 HTTP 缓存，避免旧脚本压制更新
  navigator.serviceWorker.register('/sw.js', { scope: '/', updateViaCache: 'none' })
    .then(reg => console.log('scope =', reg.scope)); // scope 决定可拦截的 URL 前缀
}

// ============ 2) sw.js ============
const CACHE = 'app-v1';                              // 缓存名即版本号，激活期按名清理
const PRECACHE = ['/', '/offline.html'];             // 只预缓存最小可离线集合

self.addEventListener('install', (e) => {
  // waitUntil 延长事件生命周期：precache 未完成则本次 install 判定失败，整体回滚
  e.waitUntil(
    caches.open(CACHE).then(c => c.addAll(PRECACHE)).then(() => self.skipWaiting())
  );
});

self.addEventListener('activate', (e) => {
  // 清理非当前版本的缓存，并立即接管已有 client（跳过 waiting）
  e.waitUntil(
    caches.keys()
      .then(keys => Promise.all(keys.filter(k => k !== CACHE).map(k => caches.delete(k))))
      .then(() => self.clients.claim())
  );
});

self.addEventListener('fetch', (e) => {
  const req = e.request;
  // 非 GET、跨域请求一律放行：不调用 respondWith 即走默认网络栈
  if (req.method !== 'GET' || new URL(req.url).origin !== self.location.origin) return;

  if (req.mode === 'navigate') {
    // 导航请求是版本入口，必须走网络，保证 HTML 引用的 hash 资源与当前发布一致
    e.respondWith(networkFirst(req, CACHE, '/offline.html'));
  } else {
    // 静态子资源按内容不可变假设处理，走 cache-first
    e.respondWith(cacheFirst(req, CACHE));
  }
});

async function cacheFirst(req, cacheName) {
  const cache = await caches.open(cacheName);
  const hit = await cache.match(req);                 // 命中时网络请求数 = 0，TTFB 由磁盘 IO 决定
  if (hit) return hit;
  const res = await fetch(req);                        // 未命中才触达网络栈（会依次命中 HTTP 缓存 → 网络）
  // 仅缓存成功且同源的完整响应；res.clone() 必需，否则 body 流被 put 消费后无法返回给页面
  if (res.ok && res.type === 'basic') cache.put(req, res.clone());
  return res;
}

async function networkFirst(req, cacheName, fallbackUrl) {
  const cache = await caches.open(cacheName);
  const ctrl = new AbortController();
  const timer = setTimeout(() => ctrl.abort(), 3000);   // 自行实现超时，否则弱网下 promise 长期 pending
  try {
    const res = await fetch(req, { signal: ctrl.signal });
    if (res.ok) cache.put(req, res.clone());            // 网络是真值源，成功即回填
    return res;
  } catch (err) {
    clearTimeout(timer);
    // respondWith 的 promise reject 会直接变成 NetworkError，必须在此显式给降级响应
    return (await cache.match(req)) || (await cache.match(fallbackUrl));
  } finally {
    clearTimeout(timer);
  }
}

// ============ 3) 验证步骤 ============
// DevTools → Application → Service Workers（确认 activated + clients.claim 生效）
// DevTools → Application → Cache Storage（观察 app-v1 的条目随请求增长/在 activate 时被清理）
// Network 面板 Size 列出现 (ServiceWorker) 即为缓存命中，无请求耗时
// 勾选 Offline：导航请求落入 networkFirst 的 catch，返回 /offline.html；hash 静态资源仍由 cacheFirst 正常返回
```

### 4. 常见误区与进阶思考
误区一：认为 respondWith 的 promise 被 reject 后浏览器会自动回落到网络。实际语义是「接管即全权负责」：promise reject 会直接向页面抛 NetworkError，降级必须由 SW 在 catch 中显式构造；同理，networkFirst 中的 fetch() 也不是「真实网络」——它仍会命中 HTTP 缓存（除非 request.cache 设为 no-store/reload 或 URL 带唯一 query），因此用 networkOnly 语义做健康探测时会得到假阳性。

误区二：认为 SW 更新会即时生效、precache 会自动跟进。实际是激活期语义：新版本停在 waiting，除非 skipWaiting() + clients.claim() 强制接管；而强制接管会制造「同一页面内旧 HTML/旧 chunk 配新 SW」的混用窗口。同时 SW 脚本自身的更新还受 updateViaCache 与浏览器强制更新检查周期约束，Cache-first 缓存的 HTML 会长期指向已删除的 hash 文件——这正是 HTML 必须走 network-first、静态资源必须 content-hash 且不可变的原因。

思考题：在「HTML 走 network-first、hash 静态资源走 cache-first、SW 用 skipWaiting 立即接管」的组合下，若用户在一个已打开的受控页面中通过 A 标签打开新页面，恰好跨越了发布边界：新页面加载到新版 HTML（引用 chunk-a2），而旧页面仍在运行旧版 JS（引用 chunk-a1）。请从 CacheStorage 的键空间设计、activate 期清理时机、以及 clients.claim 的接管粒度出发，说明如何在不放弃即时更新的前提下避免 chunk 404，并指出 stale-while-revalidate 在这套方案中会引入哪种不一致。
