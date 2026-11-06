---
title: "每日基础技术总结 · 2026-11-07 · CDN边缘节点缓存失效（Purge）的策略：即时失效与TTL刷新的权衡"
date: 2026-11-07 07:01:31
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-07 · CDN边缘节点缓存失效（Purge）的策略：即时失效与TTL刷新的权衡

## 📚 今日主题

> **CDN边缘节点缓存失效（Purge）的策略：即时失效与TTL刷新的权衡**（网络基础）

### 1. 核心概念速览
CDN 缓存失效（Purge / Invalidation）指：在对象尚未自然到达 TTL 之前，由控制面主动下发指令，使分布在各个边缘 PoP（以及中间层 shield / 父级缓存）上的某个副本立即删除（hard purge）或被标记为不可信（soft purge），从而把「源站内容变更 → 全网边缘可见新内容」的收敛窗口从「剩余 TTL」压缩到「控制面传播延迟 + 在途请求完成时间」。

本质：CDN 用「缓存键 → 多副本」换取命中率与回源成本下降，代价是内容更新时必须处理 N 个副本的失效——这是分布式缓存一致性问题的一个工程化解法。因为节点数量大、可能离线、存在在途请求与多级缓存，系统只能做到最终一致（eventual consistency），不存在原子删除。

两条正交的失效路径：
1) 被动失效（TTL 刷新）：由 Cache-Control: s-maxage / max-age / Expires 决定 freshness_lifetime，过期后由首个请求触发回源或条件重验证（If-None-Match → 304）。通信成本 O(1)（无控制面消息），但存在 stale 窗口，且过期瞬间天然产生穿透。
2) 主动失效（Purge）：控制面广播，通信成本 O(副本数)。其中 hard purge 删除条目，下一次请求必然穿透回源（击穿 / 惊群风险）；soft purge 保留 body 并置为 stale + must-revalidate，下一次请求走条件重验证，304 时仅刷新 storedAt，带宽成本接近 0。

体系位置：向上承接 HTTP 缓存语义（RFC 9111 的 freshness / validation 模型、Age 头与 Vary），向下落到分布式系统的副本失效与消息传播（失效日志、seq/watermark、at-least-once + 幂等删除）。它与浏览器 HTTP 缓存、Service Worker Cache、构建产物 contenthash 命名是同一套「新鲜度 vs 一致性 vs 回源成本」的三角权衡。

必须掌握的原因：任何把静态资源、配置、灰度规则、模型权重/分片、Prompt 模板放到 CDN 上的系统，其发布流程的正确性最终由失效语义决定。误判 purge 的语义会直接导致「配置回滚后仍持续返回旧值」「发布后部分地域长期不一致」「热点 key 硬 purge 打挂源站」这类生产事故。

### 2. 底层原理剖析
一、缓存层级与缓存键
请求路径：client → edge PoP(L1) → mid-tier shield(L2) → origin。
cache key = f(scheme, host, path, query, Vary 指定的请求头)。任何未纳入 key 但会影响表示的属性都会导致错误复用（如未对 Accept-Encoding 做 Vary 导致压缩体串味，未对 Cookie/租户头做 Vary 导致越权命中）。
freshness 判定：age = now - storedAt（跨层时需累加上游 Age 头，避免多层缓存把 TTL 语义放大）；fresh ⟺ age < freshness_lifetime，lifetime 优先级 s-maxage > max-age > Expires > 启发式。

二、TTL 刷新的机制与代价
- 惰性：没有请求就永不回源，带宽成本最低。
- 过期瞬间的并发请求会同时判定为 stale 并同时穿透，必须在边缘做 request collapsing（singleflight）：同一 key 只放行一个回源请求，其余挂起等待其结果。
- stale-while-revalidate：在 TTL 后的一段窗口内直接返回旧值，异步触发重验证，把延迟与一致性解耦。
- stale-if-error：源站 5xx / 超时时继续提供旧值，把可用性置于一致性之上。

三、Purge 的传播模型
控制面 API → 持久化失效日志（单调 seq / epoch）→ 经消息总线 fanout 到各 PoP agent → agent 本地执行 delete 或置 stale → ack 回传。
不可靠点：节点离线、网络分区、消息丢失。因此真实系统必须是「推 + 拉」结合：节点重连后用自己的 watermark 从日志回放 seq > watermark 的记录，依赖 at-least-once + 删除幂等保证最终收敛。
粒度：
- 单 URL：O(1) 消息体，精确。
- 前缀 / 通配：需要 trie 扫描全量 key 空间，代价高，多数厂商限制条数或直接降级。
- surrogate-key / cache-tag：需维护 tag → key 反向索引，用写放大与内存换「语义级精确失效」（一次业务发布只发一条 tag purge）。
多级缓存顺序：必须自内向外或全层同时失效。若只失效 edge 而未失效 shield，edge 回源会重新拿到 shield 上的旧副本，表现为「purge 无效」的假象。

四、核心竞态：purge-then-refill（旧值回填）
t0 请求 A 未命中，开始回源（源站耗时 200ms，返回旧值）
t1 源站内容更新，purge(key) 执行，条目被删除
t2 请求 A 完成，把旧 body 写回缓存 → 旧值复活，直到下次 TTL 过期
解法：per-key generation（epoch）+ 写回 CAS。回源前快照 epochAtFetch = gen[key]；写回时若 gen[key] !== epochAtFetch 则丢弃本次写入。这是无锁版本号方案，与前端并发渲染中丢弃 stale update、SW 更新中的版本切换是同一思想。注意：丢弃写入会把本次请求的成本浪费掉，后续请求会再次回源，因此必须与 singleflight 结合，避免「丢弃 → 再回源」的正反馈。

五、hard vs soft 的选型
- hard：命中率断崖式下跌，热 key 上并发请求全部穿透，形成惊群与源站雪崩；仅在「必须立刻不可见」（下架、合规、错误配置回滚）时使用。
- soft：语义是「不再信任但允许验证」。ETag 未变 → 304，边缘保留 body 只刷新 storedAt，代价近似 0；ETag 变了 → 200 全量替换。绝大多数发布场景应默认 soft。
- 工程上的最优解通常是「版本化 URL + 长 TTL」：把失效从 O(N) 广播降为 O(1) 的命名空间切换（index.html 引用 app.[contenthash].js），purge 只作用于极少数入口 HTML，旧版本文件保留以便回滚。

六、与前端已有概念的对照
- HTTP 缓存头 / ETag / 304：与边缘条件重验证完全同构；soft purge 等价于人为把条目降级为 must-revalidate。
- contenthash / ?v=xxx：即版本化 URL，用命名空间切换绕开失效广播。
- Service Worker：cache 手动管理 + install → waiting → activate 的指针原子切换，与边缘灰度的「先写新副本再切指针」一致；但 SW 是单副本、进程内、可同步判定，CDN 是多副本、跨地域、只能最终一致。
- 模块图失效（Vite/webpack HMR）：沿 importers 反向传播 invalidate，与 tag/前缀 purge 的反向索引传播同构，难点同样在「精确失效边界」的判定。
- 关键差异：CDN 存在完全不可控的下游——浏览器本地缓存（max-age 未过期时任何 purge 都无效）、企业代理、ISP 透明缓存。所以「purge 成功」从来不等于「用户看到新内容」。

### 3. 基础代码与实战验证
```text
// edge.js —— 最小 CDN 边缘节点模型：TTL 被动过期 + soft/hard purge + epoch 防回填竞态 + singleflight
// 运行：node edge.js；另起一个 8081 上的静态源站。
//   curl -s localhost:8080/a.txt                       # MISS，写入缓存
//   curl -s localhost:8080/a.txt                       # HIT
//   curl -s 'localhost:8080/purge?key=localhost:8080/a.txt&mode=soft'
//   curl -s localhost:8080/a.txt                       # 带 If-None-Match 回源，304 则复用 body
const http = require('http');

const ORIGIN = { host: '127.0.0.1', port: 8081 };
const DEFAULT_TTL = 60;   // 秒；真实系统取自 s-maxage > max-age > Expires > 启发式
const SWR = 30;           // stale-while-revalidate 窗口：TTL 之后仍可返回旧值并异步刷新

const cache = new Map();  // key -> { body, etag, storedAt, ttl, stale }
const gen = new Map();    // key -> 失效代数；单调递增，用作无锁 CAS 的版本号
let genCounter = 0;
const inflight = new Map();  // key -> Promise；singleflight，抑制过期瞬间的惊群

function keyOf(req) {
  // 缓存键必须覆盖所有影响表示的因素；此处省略 Vary 处理（真实系统会折叠进 key）
  return req.headers.host + req.url;
}

function parseTtl(cc) {
  const m = /s-maxage=(\d+)/.exec(cc || '') || /max-age=(\d+)/.exec(cc || '');
  return m ? Number(m[1]) : DEFAULT_TTL;
}

function freshness(e) {
  const age = (Date.now() - e.storedAt) / 1000;
  if (e.stale) return { state: 'stale' };              // soft purge 后强制重验证
  if (age < e.ttl) return { state: 'fresh' };
  if (age < e.ttl + SWR) return { state: 'swr' };      // 可先返回旧值，后台刷新
  return { state: 'expired' };
}

// 回源：条件请求 + 写回 CAS + singleflight
function fetchOrigin(key, path, prev, epochAtFetch) {
  if (inflight.has(key)) return inflight.get(key);      // 同一 key 只放行一个回源请求
  const p = new Promise((resolve, reject) => {
    const headers = {};
    if (prev && prev.etag) headers['If-None-Match'] = prev.etag;  // 条件重验证，命中则 304
    const req = http.request({ ...ORIGIN, path, method: 'GET', headers }, res => {
      const chunks = [];
      res.on('data', c => chunks.push(c));
      res.on('end', () => {
        // 关键不变量：写回前比对代数。若回源期间发生过 purge，gen 已推进，本次结果作废
        if ((gen.get(key) || 0) !== epochAtFetch) return resolve(prev || null);
        if (res.statusCode === 304 && prev) {
          prev.storedAt = Date.now();   // 仅刷新新鲜期，body 复用，零带宽
          prev.stale = false;
          return resolve(prev);
        }
        if (res.statusCode === 200) {
          const next = {
            body: Buffer.concat(chunks),
            etag: res.headers.etag,
            storedAt: Date.now(),
            ttl: parseTtl(res.headers['cache-control']),
            stale: false,
          };
          cache.set(key, next);
          return resolve(next);
        }
        resolve(prev || null);          // 其他状态码：保守地降级为旧值或失败
      });
    });
    req.on('error', reject);
    req.end();
  }).finally(() => inflight.delete(key));
  inflight.set(key, p);
  return p;
}

// 主动失效：先推进代数，再处理条目。顺序不可颠倒，否则仍有回填窗口
function purge(key, mode) {
  gen.set(key, ++genCounter);
  const e = cache.get(key);
  if (!e) return;                       // 未缓存也要推进代数，防止在途回源写回旧值
  if (mode === 'hard') cache.delete(key);   // 下次请求必然穿透，热 key 会惊群
  else e.stale = true;                      // 保留 body，下次请求走 304 验证
}

function send(res, e, tag) {
  res.writeHead(200, { 'Content-Type': 'text/plain', 'X-Cache': tag });
  res.end(e.body);
}

http.createServer(async (req, res) => {
  if (req.url.startsWith('/purge')) {   // 简化版控制面接口；真实系统在独立管理面 + 鉴权 + 失效日志 fanout
    const u = new URL(req.url, 'http://x');
    purge(u.searchParams.get('key'), u.searchParams.get('mode') || 'soft');
    return res.end('ok\n');             // 200 仅代表本节点已执行，不代表全网收敛
  }
  const key = keyOf(req);
  const entry = cache.get(key);
  const f = entry ? freshness(entry) : { state: 'expired' };

  if (f.state === 'fresh') return send(res, entry, 'HIT');

  if (f.state === 'swr') {              // 命中过期窗口：先返回旧值，异步重验证
    fetchOrigin(key, req.url, entry, gen.get(key) || 0).catch(() => {});
    return send(res, entry, 'HIT-STALE');
  }

  const epochAtFetch = gen.get(key) || 0;   // 回源起点的代数快照
  try {
    const fresh = await fetchOrigin(key, req.url, entry, epochAtFetch);
    if (!fresh) { res.writeHead(504); return res.end(); }
    send(res, fresh, 'MISS');
  } catch (e) {
    if (entry) return send(res, entry, 'HIT-ERROR');   // stale-if-error：可用性优先
    res.writeHead(502); res.end();
  }
}).listen(8080);
```

### 4. 常见误区与进阶思考
误区一：把 purge 当作同步、原子、强一致的删除。控制面返回 200 只表示指令已持久化并入队，收敛时间 = 消息传播 + 各 PoP 执行 + 在途请求完成，离线节点要靠 watermark 回放才能追上。更致命的是失效边界被严重低估：浏览器本地缓存（max-age 未过期时任何 purge 都无效）、企业代理、ISP 透明缓存完全不受控；多级缓存只失效了 edge 而未失效 shield 时，edge 回源会重新拉回旧副本，表现为「purge 明明成功但内容没变」。正确心智模型：purge 是一次带延迟的、最终一致的命名空间写入，必须由版本化 URL / 短 max-age 的入口文件兜底。

误区二：认为 TTL 越短越安全，或 purge 越即时越好。短 TTL 把一致性成本摊成常态化的回源 QPS 与命中率下降；硬 purge 把成本压缩成脉冲，热 key 上并发请求同时穿透形成惊群，源站瞬时被打穿。真正要评估的指标是「单位时间源站请求数 = 失效率 × 未命中请求集合大小」，而不是单看 TTL 或 purge 延迟；没有 request collapsing 时，过期瞬间的并发数会成倍放大。同理，前缀/通配 purge 的反向索引内存与全节点 fanout 成本常被低估，应优先用 tag 或版本化 URL 收窄失效范围。

思考题：请给出在「同一个 key 在任意时刻至多只有一个回源请求在途」与「任何在 purge 之前发起的回源结果都不得写入缓存」这两条约束同时成立时，系统需要维持的全部不变量（提示：至少涉及 gen 的推进顺序、gen 快照的读取时机、singleflight 的键是否应包含 gen、以及丢弃写入后如何避免回源风暴）。并进一步论证：为什么「版本化 URL + 长 TTL」在通信复杂度上严格优于「短 TTL + 主动 purge」——请从失效消息的扇出量（O(N) 广播 vs O(1) 命名空间切换）与失效窗口的可控性两个维度给出结论。
