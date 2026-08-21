---
title: "每日基础技术总结 · 2026-06-10 · HTTP Range 请求与断点续传"
date: 2026-06-10 08:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-06-10 · HTTP Range 请求与断点续传

## 📚 今日主题

> **HTTP Range 请求与断点续传**（网络基础）

### 1. 核心概念速览
HTTP Range 请求是HTTP/1.1协议定义的客户端向服务端请求资源部分字节范围的机制。其本质是通过请求头`Range: bytes=start-end`声明所需字节区间，服务端根据资源长度与范围合法性返回三种响应之一：`206 Partial Content`（部分内容）、`200 OK`（忽略Range返回完整实体）、`416 Range Not Satisfiable`（范围不可满足）。核心解决的是大文件传输中的带宽节省、并发分块下载、视频拖动播放与断点续传问题。它在网络分层中位于应用层，依赖HTTP的请求/响应模型，不依赖传输层特性。专业工程师必须掌握它，因为它是所有CDN、流媒体、下载工具、云存储的底层基础之一，前端工程师处理大文件上传下载时同样需要精确理解服务端行为与客户端状态管理。

### 2. 底层原理剖析
底层机制可分为四步：
1. 客户端发起请求，首部携带`Range: bytes=<start>-<end>`，其中end为闭区间，可省略end表示到文件末尾，可省略start（如`bytes=-500`）表示最后500字节；多段使用逗号分隔（如`bytes=0-99,200-299`）。
2. 服务端校验请求范围：首先需要资源支持Range，若不支持或语法错误，返回`200 OK`并携带完整实体，此时`Accept-Ranges`应缺失或为`none`；若范围合法，则读取对应字节流，返回`206 Partial Content`，响应头必须包含`Content-Range: bytes <start>-<end>/<total>`，其中total是资源总字节数，`Content-Length`则为实际返回的字节数（end-start+1）；若范围超出资源大小，返回`416`，响应体需包含`Content-Range: bytes */<total>`以告知可用长度。
3. 多段请求（multipart/byteranges）时，响应体需按MIME multipart格式组织，每个段用`Content-Range`标识，`Content-Type`为`multipart/byteranges; boundary=...`。
4. 客户端在续传场景中记录已接收的字节数`offset`，重发`Range: bytes=offset-`，直到服务端返回206并继续写入文件。若返回200，则需重新下载整个文件，这是对资源不可Range时的降级策略。
与前端已有知识的对比：前端熟悉的TS接口是编译期类型契约，描述数据结构形状，运行时无实体；而HTTP Range请求是运行时的字节级协议行为，通过首部字段协商传输内容。类似地，前端中`ArrayBuffer.slice`与`Blob.slice`是本地内存/文件分片操作，不涉及网络状态；而Range请求需要服务端配合，且受HTTP缓存、代理、CDN的中间节点影响，这些中间节点可能改写或忽略Range头，导致行为不一致。此外，Range请求的本质是服务端状态的确定性计算：给定资源长度L与请求范围r，决定响应状态码与字节区间，与前端函数`(L, r) => {status, body}`的纯函数逻辑高度类似，但底层涉及文件I/O、连接复用与头部校验。

### 3. 基础代码与实战验证
以Node.js原生HTTP模块实现极简Range服务器，验证核心机制：
```js
const http = require('http');
const fs = require('fs');

// 模拟一个10字节的资源内容
const content = Buffer.from('0123456789');

http.createServer((req, res) => {
  const range = req.headers.range; // 获取请求头中的Range字段

  if (!range) {
    // 无Range：返回完整内容，状态200
    res.writeHead(200, {
      'Content-Length': content.length,
      'Accept-Ranges': 'bytes'
    });
    res.end(content);
    return;
  }

  // 只支持单段bytes格式
  const match = range.match(/^bytes=(\d*)-(\d*)$/);
  if (!match) {
    // 非法Range语法，返回416并告知总长度
    res.writeHead(416, { 'Content-Range': `bytes */${content.length}` });
    res.end();
    return;
  }

  let start = match[1] === '' ? undefined : parseInt(match[1]);
  let end = match[2] === '' ? undefined : parseInt(match[2]);

  if (start === undefined && end === undefined) {
    // 空范围，非法
    res.writeHead(416, { 'Content-Range': `bytes */${content.length}` });
    res.end();
    return;
  }

  if (start === undefined) {
    // 如 bytes=-2 表示最后2个字节
    start = Math.max(content.length - end, 0);
    end = content.length - 1;
  } else if (end === undefined) {
    // 如 bytes=5- 表示从第5字节到末尾
    end = content.length - 1;
  }

  // 范围越界：start超过末尾或end<start
  if (start >= content.length || end < start) {
    res.writeHead(416, { 'Content-Range': `bytes */${content.length}` });
    res.end();
    return;
  }

  // 截取实际请求的字节区间（闭区间）
  const chunk = content.slice(start, end + 1);

  // 返回206部分内容，并携带Content-Range
  res.writeHead(206, {
    'Content-Range': `bytes ${start}-${end}/${content.length}`,
    'Content-Length': chunk.length,
    'Accept-Ranges': 'bytes'
  });
  res.end(chunk);
}).listen(3000);
```
验证命令：
- `curl -H "Range: bytes=2-5" http://localhost:3000` → 返回`2345`，状态206，Content-Range: bytes 2-5/10
- `curl -H "Range: bytes=8-" http://localhost:3000` → 返回`89`，状态206
- `curl -H "Range: bytes=-3" http://localhost:3000` → 返回`789`，状态206
- `curl -H "Range: bytes=99-100" http://localhost:3000` → 返回416，Content-Range: bytes */10

### 4. 常见误区与进阶思考
误区1：认为`Range`请求就是断点续传，只要客户端发送Range头即可。实际上服务端可能不解析Range（如某些静态文件服务器配置错误），或者中间缓存/CDN不转发Range头而直接返回完整资源；客户端必须检查状态码为206才可追加写入，否则应删除临时文件重新下载。
误区2：多段Range时误以为`Content-Length`等于所有片段长度之和。正确计算应包含multipart边界字符串、每段头部的开销，最终`Content-Length`是完整multipart实体的字节数。若使用Node的Buffer拼接时未正确计算长度，会导致响应体截断或挂起。
思考题：服务端收到`Range: bytes=0-0`，资源总长度是0字节（空文件），应返回什么状态码？`Content-Range`应如何填写？这背后反映HTTP对“空区间”与“零长度资源”的边界语义如何定义？请结合RFC 7233的区间语法推导验证。
