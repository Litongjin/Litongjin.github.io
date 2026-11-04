---
title: "每日基础技术总结 · 2026-11-05 · HTTP/1.1 Range请求在多部分响应（multipart/byteranges）中的内容协商逻辑"
date: 2026-11-05 07:02:09
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-05 · HTTP/1.1 Range请求在多部分响应（multipart/byteranges）中的内容协商逻辑

## 📚 今日主题

> **HTTP/1.1 Range请求在多部分响应（multipart/byteranges）中的内容协商逻辑**（网络基础）

### 1. 核心概念速览
HTTP/1.1 的 Range 请求是客户端在已选定的表示（representation）上请求一个或多个字节区间的机制。请求头形如 Range: bytes=0-499,1000-1999,-500；服务端先完成常规内容协商（Accept/Accept-Encoding 等）并确定当前表示及其完整长度，再判定是否接受 Range。若忽略 Range，返回 200 和完整表示；若只选出一个可满足区间且未合并，返回 206 并在顶层带 Content-Range: bytes start-end/complete-length；若选出多个区间且未合并，返回 206，Content-Type 为 multipart/byteranges; boundary=...，每个 part 各自带 Content-Range，顶层不再带单一 Content-Range；若所有区间均不可满足，返回 416 并带 Content-Range: bytes */complete-length。Accept-Ranges: bytes 只是能力通告，If-Range 用强 ETag 或 Last-Modified 做表示版本校验，避免把不同版本的字节拼接。
它位于 HTTP 语义层：在表示选择、条件请求、缓存与传输编码之间起作用。它是断点续传、媒体 seek、并行分片下载、对象存储分片读取、CDN 回源、AI 模型分片拉取的基础。专业工程师必须掌握，因为 206/multipart 的解析、缓存与代理行为直接决定字节正确性和一致性。

### 2. 底层原理剖析
Range 不是带 q 值的主动内容协商，而是对已选定表示的选择请求；服务端拥有最终裁量权，可以忽略、合并、限制数量或只满足子集。处理顺序应理解为：
1. 先完成 Accept、Accept-Encoding 等表示选择，得到 selected representation 及其完整长度 len。
2. 若存在 If-Range，且其强校验器与当前表示不匹配，则忽略 Range，返回 200 完整表示。
3. 解析 Range: bytes=range-set。区间是闭区间、零基；支持 0-499、500-、-500 三种形态。语法错误通常可忽略 Range 返回 200；语法合法但全部不可满足返回 416。
4. 规范化区间：suffix 取末尾 N 字节，open-ended 取到 len-1，end 超界截断到 len-1，start >= len 的区间丢弃。
5. 服务端可选合并重叠或相邻区间，也可保留多个 part；若最终只剩一个区间，返回单段 206 + 顶层 Content-Range；若多个区间，返回 multipart/byteranges，每个 part 独立带 Content-Range；顶层不放单一 Content-Range。
6. 若一个可满足区间都没有，返回 416 + Content-Range: bytes */len。

多段响应的线格式是 MIME multipart：每个 part 以 CRLF--boundary CRLF 开始，part 头包含 Content-Type 与 Content-Range，空行后是原始字节，part 之间以 CRLF 分隔，最终以 CRLF--boundary--CRLF 结束。boundary 由响应头 Content-Type: multipart/byteranges; boundary=... 给出，解析器必须按 MIME 边界规则处理，不能简单按字符串 split 后直接拼接。

与前端已有概念的对比：fetch/axios 发送 Range 只是设置请求头；响应必须先按 status 分支：200 表示完整表示，206 表示部分表示，416 表示不可满足。206 不保证是单个 ArrayBuffer，多段时得到的是 multipart/byteranges 容器，需要解析 part 头和边界；它和 multipart/form-data 同属 MIME multipart，但 form-data 的 part 用 Content-Disposition 描述字段名，byteranges 的 part 用 Content-Range 描述字节位置，语义完全不同。浏览器 media 元素会自动发 Range 并处理 206，但手写下载器不能假定服务端一定按请求返回。

关键一致性点：Content-Range 的完整长度是 selected representation 的长度；若存在 Content-Encoding，Range 作用于编码后的表示数据，而不是业务解码后的字节。If-Range 应使用强校验器；弱 ETag 不应用于 If-Range。206 可被缓存，但 multipart/byteranges 通常不作为完整表示直接复用；Accept-Ranges 只是能力通告，不代表每个 Range 请求都必然被满足。

### 3. 基础代码与实战验证
```text
const http = require('http');
const crypto = require('crypto');

const BODY = Buffer.from('0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ');
const CRLF = String.fromCharCode(13, 10);

// 解析 Range 头。返回 null 表示语法错误应忽略 Range；返回 [] 表示语法合法但全部不可满足。
function parseRanges(header, size) {
  if (!header || header.toLowerCase().indexOf('bytes=') !== 0) return null;
  const specs = header.slice(6).split(',').map(s => s.trim()).filter(Boolean);
  const out = [];
  for (const spec of specs) {
    const p = spec.split('-');
    if (p.length !== 2) return null;
    const sa = p[0].trim();
    const sb = p[1].trim();
    if (sa === '' && sb === '') return null;
    if (sa === '') {
      const n = Number(sb);
      if (!Number.isInteger(n) || n <= 0) return null;
      const len = Math.min(n, size);
      if (len > 0) out.push([size - len, size - 1]);
    } else if (sb === '') {
      const a = Number(sa);
      if (!Number.isInteger(a) || a < 0) return null;
      if (a < size) out.push([a, size - 1]);
    } else {
      const a = Number(sa);
      const b = Number(sb);
      if (!Number.isInteger(a) || !Number.isInteger(b) || a < 0 || b < a) return null;
      if (a < size) out.push([a, Math.min(b, size - 1)]);
    }
  }
  // 可选：合并重叠或相邻区间；服务端也可以不合并而保留多个 part。
  out.sort((x, y) => x[0] - y[0]);
  const merged = [];
  for (const r of out) {
    const last = merged[merged.length - 1];
    if (last && r[0] <= last[1] + 1) last[1] = Math.max(last[1], r[1]);
    else merged.push([r[0], r[1]]);
  }
  return merged;
}

const server = http.createServer((req, res) => {
  if (req.method !== 'GET') {
    res.writeHead(405, { Allow: 'GET' });
    return res.end();
  }
  const size = BODY.length;
  res.setHeader('Accept-Ranges', 'bytes'); // 能力通告，不保证每个 Range 都被满足
  const range = req.headers.range;
  if (!range) {
    res.writeHead(200, { 'Content-Type': 'text/plain', 'Content-Length': size });
    return res.end(BODY);
  }
  const ranges = parseRanges(range, size);
  if (ranges === null) {
    // 语法错误：本实现选择忽略 Range，返回完整表示
    res.writeHead(200, { 'Content-Type': 'text/plain', 'Content-Length': size });
    return res.end(BODY);
  }
  if (ranges.length === 0) {
    // 语法合法但全部不可满足：416 + 完整长度
    res.writeHead(416, { 'Content-Range': 'bytes */' + size });
    return res.end();
  }
  if (ranges.length === 1) {
    const s = ranges[0][0];
    const e = ranges[0][1];
    const chunk = BODY.subarray(s, e + 1);
    res.writeHead(206, {
      'Content-Type': 'text/plain',
      'Content-Range': 'bytes ' + s + '-' + e + '/' + size,
      'Content-Length': chunk.length
    });
    return res.end(chunk);
  }
  const boundary = 'B' + crypto.randomBytes(8).toString('hex');
  const parts = [];
  for (const r of ranges) {
    const s = r[0], e = r[1];
    // 每个 part 独立携带 Content-Range；顶层不再放单一 Content-Range
    parts.push(Buffer.from('--' + boundary + CRLF +
      'Content-Type: text/plain' + CRLF +
      'Content-Range: bytes ' + s + '-' + e + '/' + size + CRLF + CRLF));
    parts.push(BODY.subarray(s, e + 1));
    parts.push(Buffer.from(CRLF));
  }
  parts.push(Buffer.from('--' + boundary + '--' + CRLF));
  const payload = Buffer.concat(parts);
  res.writeHead(206, {
    'Content-Type': 'multipart/byteranges; boundary=' + boundary,
    'Content-Length': payload.length
  });
  res.end(payload);
});

server.listen(3000, () => {
  console.log('GET / with Range: bytes=0-4,10-14');
});
// 验证：curl -i -H 'Range: bytes=0-4,10-14' http://localhost:3000/
// 观察 206、Content-Type: multipart/byteranges; boundary=... 以及各 part 的 Content-Range。
```

### 4. 常见误区与进阶思考
误区一：把 206 等价于「按请求原样返回多段」，并假定 multipart/byteranges 顶层也有 Content-Range。实际服务端可忽略 Range 返回 200、可合并相邻或重叠区间、可只满足子集、可因限制拒绝；多段响应的区间与请求不保证一一对应，顶层是 multipart/byteranges，各 part 才有 Content-Range；全部不可满足是 416 + bytes */len。客户端必须先看状态码再决定解析路径。
误区二：认为 Range 作用于解码后的业务字节，或可以对 gzip 后的各 part 分别解压再拼接。Range 作用于已选定的 representation data（在 Content-Encoding 之后），Content-Range 的完整长度是该表示的长度；对压缩表示的任意字节片段单独 gunzip 通常失败，跨 part 拼接后解压才可能有效，且必须保证同一版本（If-Range 强校验器）。

思考题：某对象存储返回 Content-Encoding: gzip、ETag: abc，客户端并行发两个 Range 请求 bytes=0-99 和 bytes=100-199，均带 If-Range: abc，第二个响应却返回 200 和完整 gzip 字节。此时客户端应如何判断能否把 200 的完整字节与第一个 206 的 100 字节拼接？如果服务端返回 206 multipart/byteranges 但边界内某 part 的 Content-Range 完整长度与首个响应不一致，说明什么一致性问题？
