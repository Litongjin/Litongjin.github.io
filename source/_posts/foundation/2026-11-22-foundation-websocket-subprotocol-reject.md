---
title: "每日基础技术总结 · 2026-11-22 · WebSocket子协议（Subprotocol）协商在服务端校验失败时的报错返回"
date: 2026-11-22 07:02:29
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-22 · WebSocket子协议（Subprotocol）协商在服务端校验失败时的报错返回

## 📚 今日主题

> **WebSocket子协议（Subprotocol）协商在服务端校验失败时的报错返回**（网络基础）

### 1. 核心概念速览
本质：WebSocket 子协议是应用层协议标识符（如 json.v2、protobuf.v1），在 WS 握手期通过 Sec-WebSocket-Protocol 头完成协商，用于在连接建立前锁定升级后帧的语义与序列化格式。RFC 6455 规定：客户端在握手请求中提供逗号分隔的候选列表，服务端最多从中选定一个并在 101 响应中逐字回显；不得回显客户端未提供的 token，也不得回显多个。当服务端校验失败（候选集与服务端白名单无交集）时，存在三条本质不同的报错返回路径：(1) 握手期返回 HTTP 4xx 拒绝升级——连接从未切换到帧协议，浏览器端仅表现为 onerror + onclose(1006)，错误细节对 JS 完全不可见；(2) 完成 101 握手后立即发送 Close 控制帧（code=1002 协议错误，或应用私有区间 4000-4999 + UTF-8 reason），前端可从 CloseEvent 读到结构化错误；(3) 不回显子协议头完成握手，ws.protocol 为空字符串，由应用层判错。它解决的问题是：把协议版本/数据格式协商从数据层前置到 0-RTT 握手层，避免连接建立后才发现双方方言不通。体系位置：位于 HTTP/1.1 Upgrade 机制之上的应用层协议协商，与 TLS 的 ALPN、HTTP 的 Accept/Content-Type 内容协商同族；对 AI 工程而言，实时流式推理网关常依赖它区分 JSON/MsgPack/Protobuf 帧格式。专业工程师必须掌握的原因：握手期错误与握手后错误的可见性差异直接决定错误处理与可观测性架构设计，且这是极少数浏览器 API 刻意隐藏错误详情的场景，排查必须回到抓包与 RFC 层面。

### 2. 底层原理剖析
1. 线上协商流程（RFC 6455 §4.2.1/§4.2.2）
客户端握手请求头（HTTP 头列表语法：逗号分隔、多行可折叠合并）：
  Sec-WebSocket-Protocol: json.v2, json.v1
服务端成功路径伪代码：
  offered  = parse_list(req['Sec-WebSocket-Protocol'])
  selected = first(offered ∩ server_whitelist)           // 最多选定一个
  if selected: resp['Sec-WebSocket-Protocol'] = selected  // 逐字回显
  resp['Sec-WebSocket-Accept'] = base64(sha1(key + MAGIC_GUID))
  return 101
合法但『未协商』：不回显该头 → 握手成功，ws.protocol === ''（未选定 ≠ 失败）
非法（客户端 MUST Fail the Connection）：回显 offered 之外的 token，或回显多个 token

2. 校验失败的三条报错返回路径（服务端自主选择，对前端的可见性完全不同）
路径 A 握手期拒绝：
  不发 101，直接写普通 HTTP 错误响应（如 HTTP/1.1 400）并断连
  TCP 上的语义从未切换到 WS 帧 → 浏览器合成 onerror + onclose(code=1006, reason='')
  1006 是客户端栈的本地合成码（abnormal closure），永远不会出现在线上帧中
  JS 读不到该 HTTP 状态码与响应体——浏览器刻意不暴露握手响应细节
路径 B 握手后 Close 帧：
  先发 101（可回显也可不回显子协议），随即发送 Close 控制帧
  帧结构：字节0 = 0x88（FIN=1, opcode=0x8）；payload = 2 字节大端关闭码 + ≤123 字节 UTF-8 reason（控制帧 payload 上限 125，减去 2 字节码）
  码位语义：1002 = protocol error；4000-4999 = 应用私有区间，语义自行约定
  前端 onclose 读到 CloseEvent.code / CloseEvent.reason → 错误结构化可见
路径 C：握手成功且不回显 → 应用层用首消息判 ws.protocol === ''

3. 与前端已有知识体系的对比
  fetch/axios 体系中的 406/415 是『内容协商失败』，但 status 与 body 对 JS 完全可读；WS 握手失败被浏览器降级为黑盒 error+1006，与 CORS 失败屏蔽错误详情同属『跨层错误不透传』设计。
  前端惯用的『首消息协商』（连接后先发 {type:'hello', v:2}）是数据层协商；subprotocol 把协商前置到握手层：省一个 RTT、由协议栈强制执行，代价是失败时只能在『HTTP 拒绝（对 JS 不可见）』与『Close 帧（对 JS 可见）』之间二选一承载错误。

### 3. 基础代码与实战验证
```text
const http = require('http');
const crypto = require('crypto');

// RFC 6455 §1.3 固定 GUID，用于计算 Sec-WebSocket-Accept
const GUID = '258EAFA5-E914-47DA-95CA-C5AB0DC85B11';
const SUPPORTED = ['json.v2'];            // 服务端子协议白名单
const MODE = process.argv[2] || 'A';      // A=握手期拒绝, B=握手后Close帧

// 构造 Close 控制帧：opcode=0x8，payload = 2字节大端码 + ≤123字节 reason
function closeFrame(code, reason) {
  const payload = Buffer.alloc(2 + Buffer.byteLength(reason));
  payload.writeUInt16BE(code);            // 关闭码固定 2 字节网络序（大端）
  payload.write(reason, 2);               // UTF-8 reason，最多 123 字节
  const frame = Buffer.alloc(2 + payload.length);
  frame[0] = 0x88;                        // FIN=1 + opcode=0x8（Close 控制帧）
  frame[1] = payload.length;              // 服务端→客户端帧 MASK=0，不掩码
  payload.copy(frame, 2);
  return frame;
}

const server = http.createServer((req, res) => {
  res.writeHead(426).end();               // 普通 HTTP 请求：Upgrade Required
});

server.on('upgrade', (req, socket) => {
  // HTTP 头列表语法：逗号分隔多个候选，必须 trim
  const offered = String(req.headers['sec-websocket-protocol'] || '')
    .split(',').map(s => s.trim()).filter(Boolean);
  const selected = offered.find(p => SUPPORTED.includes(p)); // 求交集，至多一个

  // 路径 A：校验失败时拒绝完成握手——不回 101，退化为普通 HTTP 错误并断连
  if (MODE === 'A' && offered.length > 0 && !selected) {
    socket.write('HTTP/1.1 400 Bad Request\r\nConnection: close\r\n\r\n');
    return socket.destroy();              // 浏览器端只见 onerror + close(1006)
  }

  // Sec-WebSocket-Accept = base64( sha1( Sec-WebSocket-Key + GUID ) )
  const accept = crypto.createHash('sha1')
    .update(req.headers['sec-websocket-key'] + GUID).digest('base64');

  const lines = [
    'HTTP/1.1 101 Switching Protocols',
    'Upgrade: websocket',
    'Connection: Upgrade',
    'Sec-WebSocket-Accept: ' + accept
  ];
  if (selected) lines.push('Sec-WebSocket-Protocol: ' + selected); // 逐字回显，仅一个
  socket.write(lines.join('\r\n') + '\r\n\r\n'); // 写完即切换为帧协议

  // 路径 B：握手成功但立即 Close，携带前端可读的结构化错误
  if (MODE === 'B' && !selected) {
    socket.write(closeFrame(4001, 'subprotocol json.v2 required'));
    return socket.end();                  // 对端浏览器会回镜像 Close 帧
  }
});

server.listen(3000, () => console.log('listening, mode=' + MODE));

// ===== 浏览器控制台验证 =====
// 1) node ws.js A 后执行：
//    var ws = new WebSocket('ws://127.0.0.1:3000', ['chat']);
//    预期：onerror → onclose，e.code === 1006（本地合成码），e.reason === ''，
//    ws.protocol === ''——400 状态码与响应体对 JS 完全不可见
// 2) node ws.js B 后执行同样代码：
//    预期：onopen 触发后立即 onclose，e.code === 4001，
//    e.reason === 'subprotocol json.v2 required'——错误结构化可见
// 3) new WebSocket('ws://127.0.0.1:3000', ['json.v2'])：
//    预期：onopen，ws.protocol === 'json.v2'（仅当选中并回显时才非空）
```

### 4. 常见误区与进阶思考
误区 1：以为 WS 握手失败能像 fetch 一样读到 HTTP 状态码与响应体。事实上浏览器 WebSocket API 刻意不暴露握手响应的任何细节（与 CORS 错误屏蔽同源设计），握手期拒绝只表现为 onerror + onclose(1006)；且 1006 是客户端本地合成码，线上帧中永远不会出现，服务端任何主动发送 1006 的尝试都构成协议违规。推论：若业务要求前端感知拒绝原因（如『请升级客户端』），必须把该类校验失败从路径 A 移到路径 B（101 + Close(4xxx, reason)），把安全类需零信息泄露的拒绝留在握手期。
误区 2：把『不回显子协议头』当作报错。RFC 中不回显是完全合法的握手结果（未选定），连接照常建立，仅 ws.protocol === ''；唯一强制客户端 Fail 的非法情形是回显了客户端未提供的 token，或一次回显多个 token。由此派生两个次级坑：一是服务端自造协议名直接回显——浏览器立刻 Fail Connection；二是 Close reason 超过 123 字节（控制帧 payload 上限 125 减去 2 字节关闭码），轻则截断重则帧非法。
思考题：同一条『子协议不支持』错误，分别走路径 A（HTTP 400）与路径 B（101 + Close 4001）时，TCP 上的字节序列、RTT 次数、前端可观测信息各是什么？请从『101 是否发出』这一分界线推导：为什么浏览器在架构上永远无法把握手失败的 HTTP 状态码透传给 JS；若你的网关同时存在『鉴权失败（要求零信息泄露）』与『版本不兼容（要求展示升级提示）』两类校验失败，两条路径的边界应如何划分，各自的错误又应如何进入日志与监控体系？
