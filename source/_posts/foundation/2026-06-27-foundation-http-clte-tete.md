---
title: "每日基础技术总结 · 2026-06-27 · HTTP 请求走私：CL.TE 与 TE.TE 语义差异"
date: 2026-06-27 08:00:00
categories: [技术分享]
tags: ["技术分享", "安全基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-06-27 · HTTP 请求走私：CL.TE 与 TE.TE 语义差异

## 📚 今日主题

> **HTTP 请求走私：CL.TE 与 TE.TE 语义差异**（安全基础）

### 1. 核心概念速览
HTTP 请求走私（HTTP Request Smuggling）是一类利用前端代理（如反向代理、负载均衡器）与后端服务器（如应用服务器）在解析 HTTP 请求边界时的语义差异，构造恶意请求，使前端认为是一个请求，而后端却将其解析为两个或多个请求的攻击技术。CL.TE 与 TE.TE 是其中两种核心变体：CL.TE 指请求同时携带 Content-Length 和 Transfer-Encoding 头，前端按 Content-Length 解析，后端按 Transfer-Encoding 解析；TE.TE 指请求携带多个或经过混淆的 Transfer-Encoding 头，不同服务器选择不同的头作为有效指令。本质是对 HTTP/1.1 协议规范中歧义字段的利用，属于协议级安全漏洞。该知识点位于 Web 安全与网络协议交叉领域，是理解 HTTP 语义、代理链行为、以及安全防护机制（如 WAF）的重要基础。专业工程师必须掌握，因为任何涉及代理转发、请求解析的架构都可能遭受此攻击，且它往往能绕过传统权限控制、缓存投毒等，导致严重安全事件。理解它需要深入协议解析的底层逻辑，而非仅停留在 API 调用层面。

### 2. 底层原理剖析
HTTP/1.1 规范（RFC 7230）规定，当请求同时包含 Content-Length 与 Transfer-Encoding 时，必须忽略 Content-Length，仅采用 Transfer-Encoding 来确定消息体长度。但现实实现中，不同服务器对这两个头的解析顺序、容错程度、优先级处理存在差异。CL.TE 攻击的原理是：前端服务器严格按 Content-Length 读取请求体，而后端服务器则遵循 Transfer-Encoding（通常为 chunked）。攻击者构造一个请求，其中 Content-Length 指示一个较短的请求体，而 Transfer-Encoding 使用 chunked 编码，在 chunked 流中提前终止（以 0 结尾），然后在 chunked 流后面附带真正的恶意请求。前端按 CL 只读取了第一部分，认为请求已结束，便将其整体转发给后端；后端按 TE 解析，读取完 chunked 流后，发现还有剩余字节，会将剩余字节作为下一个请求的开始，从而走私成功。TE.TE 攻击则是利用不同服务器对 Transfer-Encoding 头字段的解析差异：例如某些服务器对大小写敏感（Transfer-Encoding vs transfer-encoding）、忽略额外空白、或只取第一个头，而另一些则取最后一个或合并多个头。攻击者发送多个 Transfer-Encoding 头，其中一个被前端视为有效（如 'chunked'），另一个被后端视为有效（如 'identity' 或故意拼写错误），导致边界判断不一致。

与前端工程师熟知的概念对比：这类似于 TypeScript 接口与 Java 接口的差异——两者都定义了契约，但 TypeScript 接口是结构化的（编译期检查），Java 接口是行为化的（运行期多态），对相同源码的语义解读不同。同样，HTTP 请求走私利用的是不同 HTTP 实现者对相同报文的不同语义解读。前端中一个典型类比是跨浏览器事件处理差异：不同浏览器对同一 HTML 的解析规则不同（如未闭合标签的处理），攻击者利用这种差异注入内容。但更精确地，它类似于在双缓存系统中，一个进程按行读取，另一个按长度读取，攻击者构造的数据使两种解析器产生不同的边界划分。

### 3. 基础代码与实战验证
以下使用 Python 原始 socket 构造一个 CL.TE 请求走私的演示。假设前端代理按 Content-Length 解析，后端按 Transfer-Encoding (chunked) 解析。攻击者发送一个看似正常的 POST 请求，但 CL 指示 body 长度为 4（'0\r\n\r\n' 的前 4 字节），而实际 body 中 chunked 流结束后紧接一个走私的 GET 请求。

```python
import socket

# 目标：前端代理（假设监听 8080）与后端服务器（假设监听 80）
# 构建原始请求
request = (
    "POST / HTTP/1.1\r\n"
    "Host: target.com\r\n"
    "Content-Length: 4\r\n"          # 前端按此长度读取 body，即 '0\r\n'
    "Transfer-Encoding: chunked\r\n" # 后端按此编码解析 body
    "\r\n"
    "0\r\n"                            # chunked 流终止标志
    "\r\n"
    "GET /admin HTTP/1.1\r\n"          # 走私的请求，前端认为已结束，后端将其视为新请求
    "Host: target.com\r\n"
    "\r\n"
)

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("localhost", 8080))  # 连接前端代理
s.sendall(request.encode())
# 接收响应（略）
s.close()
```

关键解释：
- `Content-Length: 4`：前端会读取 body 的前 4 字节，即 `'0\r\n'`（0 是 chunked 终止块长度，后面跟 CRLF）。前端认为请求结束，不再读取后续字节。
- `Transfer-Encoding: chunked`：后端看到此头，使用 chunked 解码，读取到 `0\r\n` 后认为 chunked 流结束。但之后还有 `\r\n` 以及 `GET /admin ...`，后端会将这些剩余数据视为连接上接下来的新请求，从而执行了走私的请求。
- 实际环境中需要根据前端和后端的解析细节调整，但核心是利用两个头的优先级差异。

### 4. 常见误区与进阶思考
常见误区 1：认为现代 HTTP 服务器和代理已经完全修复了此类漏洞。实际上，虽然主流服务器（如 Apache、Nginx）对标准 CL.TE 有防御，但各种自定义代理、微服务网关、CDN 或旧版组件仍存在解析差异，且 TE.TE 的混淆方式层出不穷（如大小写、多余空格、重复头等），难以全面封堵。工程师应始终对任意来源的 HTTP 请求头进行严格校验，并在代理层统一规范。

常见误区 2：混淆 CL.TE 和 TE.CL。CL.TE 是前端用 Content-Length，后端用 Transfer-Encoding；TE.CL 则相反。两者利用的解析器顺序不同，攻击构造也完全不同。理解时必须明确哪个环节使用哪个头，否则无法正确构造或防御。

思考题：假设前端代理使用 Transfer-Encoding 解析请求体，但只识别小写 'transfer-encoding'；后端使用 Content-Length 解析，且会忽略所有 Transfer-Encoding 头。如何构造一个 TE.CL 攻击请求？请设计具体报文，并说明前端与后端分别如何解析，最终走私的请求是什么？
