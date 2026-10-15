---
title: "每日基础技术总结 · 2026-10-16 · 负载均衡器七层会话保持（Session Affinity）的实现原理与Cookie注入陷阱"
date: 2026-10-16 07:01:08
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-16 · 负载均衡器七层会话保持（Session Affinity）的实现原理与Cookie注入陷阱

## 📚 今日主题

> **负载均衡器七层会话保持（Session Affinity）的实现原理与Cookie注入陷阱**（网络基础）

### 1. 核心概念速览
七层会话保持指工作在 HTTP 应用层的负载均衡器，通过解析与改写 HTTP 头部，把同一客户端的连续请求固定路由到同一后端实例的机制。它解决的问题是：当应用把会话状态存放在后端进程内存（如 JSESSIONID 对应的容器内 HttpSession）时，无状态的轮询/最少连接调度会把同一会话的请求打散到不同实例，导致状态丢失。其本质是『以 Cookie 作为路由向量』：LB 在首次响应中注入一条编码了后端标识的 Set-Cookie，浏览器后续请求自动回带该 Cookie，LB 解析后绕过调度算法直连目标后端。它在体系中的位置：与一致性哈希、健康检查、连接复用同属反向代理/CDN/API 网关这类七层基础设施的核心调度层，是『有状态服务水平扩展』这一分布式基本矛盾的直接产物。专业工程师必须掌握它，因为它是 HTTP 头部改写、Cookie 作用域语义、共享缓存安全三个领域的交叉节点——恰好位于前后端边界上最易出错的位置，且故障表现为偶发、难复现。

### 2. 底层原理剖析
一、四层与七层亲和的本质差异
- 四层：基于 IP 五元组做源 IP 哈希（source-ip hash → backend）。LB 不解析应用层，仅见 IP:port。NAT/运营商 CGNAT/企业代理之后，成千上万用户共享一个源 IP，哈希会把整个出口绑定到单台后端——亲和粒度错误，负载严重倾斜。
- 七层：LB 完整解析 HTTP 头，以 Cookie 为路由 key。亲和粒度从『网络出口』细化到『单个浏览器实例』，且与网络拓扑完全解耦。

二、Cookie 注入的两种实现
1. 被动学习：应用自己下发会话 Cookie（如 JSESSIONID=abc），LB 旁路观察后端响应并记录 abc → backend-2 的映射，之后见到该值即定向转发。要求 LB 先『见到』该响应才能学习，冷启动存在学习窗口。
2. 主动注入（主流）：首次请求无亲和 Cookie → LB 走调度算法选后端 → 响应回程时 LB 向响应的 Set-Cookie 头数组追加一条亲和条目（HAProxy 的 cookie insert、AWS ALB 的 AWSELB、Azure 的 ApplicationGatewayAffinity）。

三、令牌编码的两条路线
- 无状态编码：Cookie 值可直接推导出目标后端（明文 server id，或一致性哈希环的虚拟节点序号）。LB 零内部状态、节点可水平扩展；代价是明文泄露后端拓扑，且后端池扩缩容引起映射大面积失效。
- 有状态映射：Cookie 值是不透明令牌，LB 内部维护 token → backend 表（AWS ALB 即此路线）。不泄露拓扑、后端池变更不必然失效；代价是 LB 需要状态同步（双活集群需共享/复制映射），LB 重启或换节点时令牌全体失效。

四、完整请求路径（文字流程）
1. 请求到达 → 解析 Cookie 头（RFC 6265：多个 cookie 以 '; ' 拼接成单条请求头）→ 提取亲和键；
2. 查映射：命中且后端健康 → 绕过调度器直连转发；
3. 未命中/后端下线 → 调度算法重选后端，并标记本响应需要注入；
4. 响应回程：向 set-cookie 头数组 push 亲和条目，同时注入 Cache-Control: no-store（防止共享缓存存储含 Set-Cookie 的响应，即 CDN 缓存污染）；
5. indirect 语义（HAProxy 的 indirect 选项）：转发给后端前从 Cookie 头剥离亲和键——应用层对 LB 的存在完全无感知，亲和 Cookie 是纯控制平面的旁路数据。

五、与前端既有知识体系的对接
- 视角差异：前端把 Cookie 当『应用状态载体』，其回带由 Domain/Path/SameSite/Secure 裁决；L7 LB 把同一套属性当作『路由控制平面』复用。SameSite=Lax 下跨站 POST 不携带亲和 Cookie → 亲和断裂 → 请求被重新调度到另一实例 → 内存 session 丢失。LB 无法控制浏览器的 Cookie 裁决行为，这是该机制的结构性盲区。
- 头部不对称性：请求头 Cookie 是多条合一（'; ' 拼接的单条头）；响应头 Set-Cookie 每个 cookie 必须独立成条、严禁合并——合并后浏览器只解析第一条。这正是 Node 18+/浏览器 fetch 规范新增 Headers.getSetCookie() 的动因：get('set-cookie') 会把多条 join 掉。写惯了 headers.get() 的人在手写代理时极易在此出错。

### 3. 基础代码与实战验证
```text
// 纯 Node 内置模块实现七层 Cookie 亲和：node affinity-lb.js
const http = require('http');

// ---- 后端池：每个实例持有『内存态』（进程内计数器），用于验证亲和 ----
const backends = [{ id: 'b1', port: 9001, n: 0 }, { id: 'b2', port: 9002, n: 0 }];
backends.forEach(b => http.createServer((req, res) => {
  b.n++;                                 // 内存状态：若亲和失效，请求会被打散到别处
  res.end(`backend=${b.id} hits=${b.n}`);
}).listen(b.port));

// ---- LB 侧：有状态映射 token -> backend.id ----
const map = new Map();
let rr = 0;

http.createServer((creq, cres) => {
  // ① 解析请求 Cookie 头：RFC6265 规定单条拼接格式 'k1=v1; k2=v2'
  const jar = Object.fromEntries((creq.headers.cookie || '')
    .split(';').map(s => s.trim().split('=')));
  let backend = jar.AFF ? backends.find(b => b.id === map.get(jar.AFF)) : null;
  const needInject = !backend;
  if (needInject) backend = backends[rr++ % backends.length]; // ② 无令牌/令牌失效 → 调度算法选路

  // ③ indirect 语义：剥离亲和键再转发，应用层永远看不到 LB 的私有 Cookie
  const fwd = (creq.headers.cookie || '').split(';')
    .map(s => s.trim()).filter(s => s && !s.startsWith('AFF=')).join('; ');

  const up = http.request({
    host: '127.0.0.1', port: backend.port,
    method: creq.method, path: creq.url,
    headers: { ...creq.headers, host: 'app.internal', cookie: fwd || undefined }
  }, ures => {
    // ④ set-cookie 必须保持数组逐条下发：join 成字符串会让浏览器只解析第一条
    const setCookies = [].concat(ures.headers['set-cookie'] || []);
    if (needInject) {
      const token = backend.id + '.' + Math.random().toString(36).slice(2, 10);
      map.set(token, backend.id);       // 不透明令牌：Cookie 值本身不泄露后端拓扑
      setCookies.push(`AFF=${token}; Path=/; HttpOnly; SameSite=Lax; Max-Age=3600`);
    }
    // ⑤ 防 Cookie 注入陷阱：禁止共享缓存存储含 Set-Cookie 的响应，避免 CDN 污染
    cres.writeHead(ures.statusCode, {
      ...ures.headers, 'set-cookie': setCookies, 'cache-control': 'no-store'
    });
    ures.pipe(cres);
  });
  creq.pipe(up);
}).listen(8080);

// 验证：
// for i in $(seq 1 5); do curl -s -c jar.txt -b jar.txt http://127.0.0.1:8080/; echo; done
// 五次输出同一 backend=bx 且 hits 递增 → 亲和成立
// rm jar.txt 后重跑 → 被调度到另一实例；重启本进程 → map 清空 → 所有旧令牌失效（有状态方案的固有失效边界）
```

### 4. 常见误区与进阶思考
误区一：把会话保持当成高可用手段。Affinity 在故障场景下反而放大伤害：绑定实例宕机时其上所有内存 session 一次性丢失，LB 只能重调度并覆盖旧 Cookie——受影响用户被批量『登出』。Affinity 的本质是『拿故障域换状态局部性』；正确演进方向是 session 外置（Redis 等集中存储）使任意实例可服务任意请求，亲和退化为本地缓存命中率的优化项，而非正确性依赖。

误区二：认为注入了 Cookie 亲和就必然生效。Cookie 是否回带由浏览器按 Domain/Path/SameSite/Secure 裁决，LB 无法控制客户端行为：SameSite=Lax 下跨站 POST（支付回调、跨站表单提交）不携带亲和 Cookie → 静默重调度 → 内存 session 丢失，线上表现为偶发、难复现的登录态异常；Safari ITP/隐私模式清理 Cookie、部分非浏览器 HTTP 客户端忽略 302 响应中的 Set-Cookie，同样是亲和链路上的静默断裂点。代码层陷阱：任何中间层把 set-cookie 合并成单条字符串，浏览器只解析第一条——应用的 Set-Cookie 与 LB 注入的亲和 Cookie 互相覆盖，症状是『亲和时灵时不灵』。

思考题：设 LB 自身为双活集群（上游四层做 DNAT 轮询，同一客户端的连续请求可能落在不同 LB 节点）。请分析：本文实现的『有状态映射表』方案此时会发生什么？若改用『无状态编码』（Cookie 值 = 一致性哈希环上目标后端的虚拟节点），后端池从 2 台扩到 3 台时旧 Cookie 的映射如何退化、退化比例约是多少？两种方案分别把失效边界放在『LB 状态丢失』与『后端池拓扑变更』处——这如何解释 AWS ALB 选择不透明令牌 + 集中解析、而 Nginx sticky 模块默认明文 server id 的工程取舍？
