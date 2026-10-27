---
title: "每日基础技术总结 · 2026-10-28 · HTTP/2 PUSH_PROMISE服务的浏览器拦截策略与被动推送的风险权衡"
date: 2026-10-28 07:03:24
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-28 · HTTP/2 PUSH_PROMISE服务的浏览器拦截策略与被动推送的风险权衡

## 📚 今日主题

> **HTTP/2 PUSH_PROMISE服务的浏览器拦截策略与被动推送的风险权衡**（网络基础）

### 1. 核心概念速览
**定义**：PUSH_PROMISE 是 HTTP/2（RFC 7540 §6.6、§8.2）的帧类型 0x5，只能由服务端在对端（客户端）发起的 stream 上发送，帧 payload 为 `R(1bit) | Promised Stream ID(31bit) | Header Block Fragment`，头块经 HPACK 压缩并携带 END_HEADERS 标志，其语义等价于一次完整请求（伪头 `:method`/`:path`/`:scheme`/`:authority`）。它宣告：服务端将在一个新的、由服务端发起的偶数 stream 上直接下发该请求的响应；客户端即使从未请求该资源，也必须让该 stream 进入 `reserved(remote) → half-closed(remote)` 状态机，并且不得在该流上发送请求。

**解决什么问题**：资源发现延迟（discovery latency）。HTTP 的依赖图（HTML→CSS→字体、JS→动态 import()、模块预加载）只有在客户端开始解析响应体后才能建立，push 试图把“发现”这一环节上移到服务端，从而省掉一个 RTT 的发现窗口，并使依赖资源与父响应共享同一连接和同一拥塞窗口并行传输。

**机制要点**：①帧写在父 stream（奇数 ID）上，实际数据走新建的偶数 stream；②通过 SETTINGS_ENABLE_PUSH(0x2) 协商，通过 RST_STREAM(CANCEL) 逐条拒绝；③推送资源受连接级流量控制与 MAX_CONCURRENT_STREAMS 约束。

**在体系中的位置**：应用层传输优化，是 HTTP/2 多路复用（stream 抽象）中唯一的“服务端主动”能力，与 HPACK、流量控制、优先级（RFC 7540 依赖树，后被 RFC 9218 Extensible Priorities 取代）构成四个正交机制。它和 CDN 预热、`<link rel=preload>`、103 Early Hints、Priority Hints 处在同一决策空间，争夺同一个目标：缩短关键渲染路径。

**为什么必须掌握**：它是“资源发现权归属”这一架构命题最清晰的案例：服务端在**预测**，客户端在**观测**。2022 年 Chrome 106 移除该能力，Firefox 长期默认关闭，Safari 从未实现，HTTP/2 Push 已不再可用；但其失败原因（客户端缓存状态不可见、已发送字节不可撤回、与关键路径共享 TCP 拥塞窗口）正是现代性能优化机制（preload、Early Hints、fetchpriority）的设计依据。不理解它，就无法解释浏览器为何系统性拒绝把预加载决策权交给服务端。

### 2. 底层原理剖析
**一、线格式与状态机约束**
- 帧头固定 9 字节：Length(24) | Type(8)=0x5 | Flags(8) | R(1)+StreamID(31)。
- 合法发送方：仅服务端。合法承载 stream：仅对端发起的 stream。
- Promised Stream ID 必须为偶数、且大于该连接上已出现的任何 stream ID（严格单调递增），否则触发连接级 PROTOCOL_ERROR。
- 客户端接收后为 promised stream 建立本地状态 `reserved(remote)`，可立即 RST_STREAM(CANCEL=0x8) 撤销，此后该流不再接受 HEADERS/DATA。

**二、单连接时间线**
```
C → HEADERS(:path=/)            streamID=1
S → PUSH_PROMISE(streamID=1)    promised=2   // 客户端此时才得知 /app.css 存在
S → HEADERS(:status=200)        streamID=2
S → DATA(...)                   streamID=2 → END_STREAM
C → RST_STREAM(CANCEL)          streamID=2   // 若拒绝，只能阻断后续帧
C → WINDOW_UPDATE(conn)                      // 连接级窗口
```

**三、控制面**
- `SETTINGS_ENABLE_PUSH(0x2)`：默认值 **1**。客户端在连接前言后的首个 SETTINGS 中置 0，服务端再发 PUSH_PROMISE 即为连接级 PROTOCOL_ERROR。
- `SETTINGS_MAX_CONCURRENT_STREAMS` 决定了可以并发推送多少条 stream；Chrome 曾硬性限制每连接 100 条推送。
- `RST_STREAM(CANCEL)` 是唯一逐条拒绝手段。
- `Cache-Digest`（draft-kazuho-h2-cache-digest，未标准化）曾是让服务端感知客户端缓存的唯一机制：Chrome 实现后又移除，原因是在协商成本、摘要过期、跨站缓存探测隐私面三方面都不可接受。这是 HTTP/2 Push 失败的核心技术原因：**信息公开面不对称**。

**四、浏览器拦截立场（事实层面的结果）**
- Chromium：M106（2022-09）移除 PUSH_PROMISE 接收与 push 缓存；移除原因可归纳为：使用率 <0.05%，收益被 preload/Early Hints 覆盖，重复传输率高，实现复杂度与安全面（跨源 push 需证书覆盖、共享缓存投毒）不划算。
- Firefox：`network.http.http2.enable-push` 长期为 false。
- Safari：从未实现。
- 通用拦截策略模型（浏览器实际行为）：①解析到 PUSH_PROMISE 后先查连接级 push cache 与 HTTP 缓存 key；②命中或超出配额即 RST_STREAM；③已进入 socket 发送队列的字节无法回收，因此拒绝只对**后续** DATA 生效；④跨源推送若证书不覆盖 `:authority`，直接按安全错误拒绝。

**五、被动推送的风险权衡（为什么是“被动”）**
1. **不可撤回的下注**：PUSH_PROMISE 之后服务端通常立即写入 DATA，客户端 RST 到达前字节已经在途；成本是 `已传输字节`，不是 `被拒次数`。
2. **缓存状态不可见**：服务端不知道 HTTP 缓存、Service Worker Cache、内存缓存，也不知道多标签页与二次导航。70% 的命中率意味着 70% 的推送是冗余传输。
3. **拥塞窗口共享**：push stream 与首屏关键资源共用一条 TCP 连接及其 cwnd；低优先级推送直接挤占 LCP 字节。
4. **优先级不可表达**：RFC 7540 的依赖树在实践中失效，RFC 9218 未给 push stream 定义有意义的语义，服务端无法表达“推 CSS 但别挡首屏”。
5. **中间盒不透明**：不理解 push 的代理/网关可能错误转发、断开或重复注入。

**六、与前端已有概念的映射**
- 与 `<link rel=preload>` 对比：preload 由客户端**解析到引用之后**发起，能命中 HTTP 缓存或 Service Worker 而直接短路，不产生网络字节；push 跳过缓存校验，落在独立的连接级 push cache，且仅在连接生命周期内有效。差异本质是**信息优势方在谁手里**。
- 与 103 Early Hints 对比：Early Hints 用普通 HEADERS 帧发送 Link 头，不下注、不占用新 stream、不污染 HPACK 动态表；客户端收到 hint 后仍用自己的缓存与优先级策略决定是否 preload。它是“服务端提供线索、客户端决策”的最优折中点。
- 与 JS 的静态 `import` 和动态 `import()` 对比：push 相当于构建期把依赖静态内联进响应，客户端无法基于运行时状态（缓存命中、媒体查询、导航上下文、条件加载）做裁剪；preload 与动态 import 则把裁剪权留给运行时。

### 3. 基础代码与实战验证
```text
以下为最小可验证工程（Node 18+ 内置 node:http2，无需第三方依赖）。

1) 生成自签证书：
openssl req -x509 -newkey rsa:2048 -nodes -keyout key.pem -out cert.pem -days 1 -subj '/CN=localhost'

2) server.js：
const http2 = require('node:http2');
const fs = require('node:fs');

const server = http2.createSecureServer({
  key: fs.readFileSync('key.pem'),
  cert: fs.readFileSync('cert.pem'),
});

server.on('stream', (stream, headers) => {
  if (headers[':path'] !== '/') return;
  stream.respond({ ':status': 200, 'content-type': 'text/html' });
  stream.end('<link rel="stylesheet" href="/app.css"><h1>hi</h1>');

  // pushStream 内部构造 PUSH_PROMISE 帧：R(1) + promisedStreamId(31，偶数) + HPACK 头块
  // 该帧写入父 stream（客户端发起的奇数 stream），Flags 带 END_HEADERS
  stream.pushStream({ ':method': 'GET', ':path': '/app.css' }, (err, pushStream) => {
    if (err) return; // 对端 SETTINGS_ENABLE_PUSH=0 时此处报错，说明协商在生效
    pushStream.respond({ ':status': 200, 'content-type': 'text/css' });
    // 此处的 DATA 已进入连接级发送队列，与首屏字节共享 TCP cwnd；
    // 即使客户端随后 RST_STREAM，这些字节也已计入传输成本
    pushStream.end('body{margin:0}');
  });
});
server.listen(8443);

3) client.js（观察并主动拒绝）：
const http2 = require('node:http2');
const client = http2.connect('https://localhost:8443', { rejectUnauthorized: false });

// remoteSettings 反映服务端侧配置；客户端若要禁用，需在 connect 选项 settings 中置 enablePush:false
client.on('remoteSettings', (s) => console.log('server ENABLE_PUSH =', s.enablePush));

// 服务端发起的偶数 stream 到达，requestHeaders 即 PUSH_PROMISE 携带的请求头块
client.on('stream', (pushedStream, requestHeaders) => {
  console.log('promised:', requestHeaders[':path'], 'streamId:', pushedStream.id);
  // 模拟浏览器拦截：只能阻断后续 DATA，已入队/在途的 PUSH_PROMISE 与部分 DATA 不可回收
  pushedStream.close(http2.constants.NGHTTP2_CANCEL);
});

const req = client.request({ ':path': '/' });
req.setEncoding('utf8');
req.on('data', (d) => process.stdout.write(d));
req.on('end', () => client.close());

4) 验证点：
- 运行后在 client 端看到 `promised: /app.css` 与偶数 streamId，证明 PUSH_PROMISE 的 promised id 分配规则。
- 在 connect 选项中加 `settings: { enablePush: false }` 重跑，服务端 pushStream 回调返回错误，证明 SETTINGS_ENABLE_PUSH 的协商语义。
- 用 wireshark 或 chrome://net-export 抓包，确认帧顺序为 HEADERS(odd) → PUSH_PROMISE → HEADERS(even) → DATA，且 PUSH_PROMISE 的 Flags 含 END_HEADERS。
```

### 4. 常见误区与进阶思考
**误区一：认为“拒绝推送”等于零成本。** RST_STREAM(CANCEL) 与 SETTINGS_ENABLE_PUSH=0 只能阻止**尚未发送**的帧；PUSH_PROMISE 本身以及紧随其后的 DATA 一旦写入 socket 发送缓冲区，就消耗了带宽与拥塞窗口。更隐蔽的成本是 HPACK 动态表被污染：被推送请求的头块写入编码器动态表后，会改变后续帧的索引编号，使后续头部的压缩表示偏离双方最优。真正的成本度量单位是“已传输字节”，而非“被拒绝的推送条数”。

**误区二：认为 HTTP/2 Push 只是 preload 的服务端版本，区别仅在于发起方。** 二者真正的区别在于**信息面**，具体如下：客户端掌握 HTTP 缓存、Service Worker 缓存、导航上下文、媒体查询、fetchpriority、连接复用情况；服务端掌握的是历史统计模型（“这条路径常引用哪些资源”）。另外，push 落在独立的连接级 push cache 中，绕过 HTTP 缓存的验证语义，可能把服务端陈旧内容下发；跨文档导航时 push cache 完全失效。因果不可逆：信息劣势方下注，冗余不可消除。

**附带误区：SETTINGS_ENABLE_PUSH 默认为 0。** 默认值是 1，必须由客户端显式关闭；这也是历史上大量站点在未协商的情况下收到推送的根因。

**思考题：** 设某站 `/` 的 HTML 在 300ms 内引用 `/app.css`，服务端在收到 `/` 请求的瞬间即可确定该依赖；用户二次访问时 `/app.css` 的 HTTP 缓存命中率为 p。请分别对 PUSH_PROMISE 与 103 Early Hints 建模 100 次导航的**期望额外字节成本**（公式需包含：push 场景下命中缓存时仍会重复传输的 CSS 体积 `S_css`、HPACK 头块开销 `H`、RST 窗口内未撤回的在途字节占比 α；Early Hints 场景下 hint 帧开销与其命中缓存时的短路行为），并推导出在“push stream 与首屏字节共享同一 TCP 拥塞窗口”这一约束下，p 超过何值时 push 的净收益为负。进一步论证：为何即使 p 很小（缓存命中率极低），Chrome 仍然选择移除 push，而不是保留并调优？请从可观测性、可取消性、优先级表达力三个维度作答。
