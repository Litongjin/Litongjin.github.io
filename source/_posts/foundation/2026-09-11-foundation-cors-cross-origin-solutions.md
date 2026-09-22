---
title: "每日基础技术总结 · 2026-09-11 · 跨域的本质与解决方案"
date: 2026-09-11 18:32:46
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-11 · 跨域的本质与解决方案

## 📚 今日主题

> **跨域的本质与解决方案**（前端底层与计算机基础）

### 1. 核心概念速览
跨域问题的本质是浏览器同源策略（Same-Origin Policy）对 JS 程序"读取跨源资源"能力的约束。同源的定义是协议、主机、端口三者完全一致，任一不同即跨源。必须澄清的底层机制是：SOP 限制的是"读取"而非"发送"——HTML 平台本身允许 img/script/form 等元素无条件跨源发送请求，但 JS 通过 XMLHttpRequest/fetch 发出的请求，其响应若未经服务端显式授权，浏览器会在网络栈与 JS 堆栈的交界处将响应体剥离，使 JS 侧仅得到一个类型化的异常（TypeError），而底层 TCP/HTTP 事务可能已经完整完成。CORS（Cross-Origin Resource Sharing）是 Web 平台标准化的跨源授权协商协议：服务端通过特定 HTTP 响应头向浏览器声明许可策略，浏览器作为执行环境负责最终裁决。它在整个计算机体系中的位置属于"浏览器内核安全模型"，是介于网络协议栈（HTTP/TCP/IP）与 JS 引擎之间的一道客户端强制（client-enforced）访问控制层，而非传输层协议，也非服务端安全机制。专业工程师必须掌握它：因为它是前后端分离、微服务、开放平台等架构下一切跨源调用的默认约束，是理解 CSRF 攻击链、Credentials 传递策略、网关鉴权与浏览器策略之间边界的基础。能否准确区分"谁在执行策略"是页面工程师与系统工程师的认知分水岭。

### 2. 底层原理剖析
核心机制拆解：
1. 浏览器在发送任意跨源请求时，自动附加 Origin 头，其值恒为当前页面的源（scheme+host+port），由浏览器保护，JS 无法篡改。同源请求则不携带 Origin。
2. 响应到达浏览器后，浏览器将响应头的 Access-Control-Allow-Origin 与当前页面源做精确匹配（允许 * 通配的例外场景见误区）。不匹配或缺失，则响应体不进入 JS 上下文。
3. 请求被分为两类，拦截时机截然不同：
   - 简单请求（simple request）：方法为 GET/HEAD/POST，且仅使用表单可携带的 Content-Type（application/x-www-form-urlencoded、multipart/form-data、text/plain），无自定义头。这类请求直接发出，拦截发生在"响应读取"阶段——请求与业务副作用已经落在服务端。
   - 预检请求（preflight）：任何携带自定义头、使用非上述 Content-Type、或使用 PUT/DELETE/PATCH 等方法的请求。浏览器先发出一个不带业务载荷的 OPTIONS 请求，附 Access-Control-Request-Method 与 Access-Control-Request-Headers 两个头，询问服务端是否允许真实请求；服务端以 Access-Control-Allow-Methods、Access-Control-Allow-Headers、Access-Control-Allow-Origin 回应。校验失败则真实请求永不发出。

浏览器裁决算法精确表述：
```
function corsGate(origin, request):
    if request is simple:
        response = networkSend(request)          // 请求已出网，服务端副作用已执行
        if allowOrigin(response) matches origin:
            return responseBody to JS
        else:
            throw CORSReadBlocked                // 响应体被浏览器剥夺，JS 看到 TypeError
    else:
        preflightResponse = networkSend(OPTIONS,
            Origin: origin,
            Access-Control-Request-Method: request.method,
            Access-Control-Request-Headers: request.customHeaders)
        if not (allowOrigin(preflightResponse) matches origin
            and allowMethods contains request.method
            and allowHeaders contains every customHeader):
            throw CORSPreflightFailed            // 真实请求零发送
        response = networkSend(request)
        if allowOrigin(response) matches origin:
            return responseBody to JS
        else:
            throw CORSReadBlocked
```

与前端已有知识的类比：Java 的接口是编译期与运行期同时存在的类型契约，由 JVM 在类加载/链接阶段强制校验，违反契约直接导致链接失败或运行时异常；TypeScript 的接口是纯编译期的结构化类型约束，编译产物中零残留，运行时根本不存在该实体。CORS 头的本质与 TS 接口一致——它只是 HTTP 里两个普通头字段，在服务端、代理、网络层没有任何天然语义可言；只有当请求流经"浏览器这个执行环境"时，这些头才被读取并产生裁决效力。curl、Node 服务端调用完全无视这些头，正如 JS 运行时完全不认识 TS 接口。理解"契约只有存在于某个有解释权的边界内才生效"，是这两组概念共通的底层逻辑。

### 3. 基础代码与实战验证
```text
用 Node 原生 http 模块构造极简跨源服务，不依赖任何框架，直接验证 CORS 头与浏览器裁决行为：

// server.js —— 监听 8080 的跨源接口
const http = require('http');

http.createServer((req, res) => {
  // 浏览器跨源请求必带 Origin 头；同源请求与 curl 请求均不带
  const origin = req.headers.origin || '';

  // OPTIONS 即预检分支：浏览器发现非简单请求时自动触发
  if (req.method === 'OPTIONS') {
    res.writeHead(204, {
      'Access-Control-Allow-Origin': origin,          // 回显该源，表示授权
      'Access-Control-Allow-Methods': 'GET, PUT',     // 声明后续真实请求允许的方法
      'Access-Control-Allow-Headers': 'Content-Type'  // 声明允许携带的非简单请求头
    });
    res.end();
    return;
  }

  // 简单请求或预检通过后的真实请求
  if (req.method === 'GET') {
    // 删除此头之后，请求仍到达此处（服务端日志可证），但浏览器端 JS 读不到响应体
    res.writeHead(200, { 'Access-Control-Allow-Origin': origin });
    res.end('plain-text');
    return;
  }

  if (req.method === 'PUT') {
    res.writeHead(200, { 'Access-Control-Allow-Origin': origin });
    res.end('put-ok');
  }

  res.writeHead(405);
  res.end();
}).listen(8080);

// 浏览器控制台验证（需从另一个源，如 http://localhost:5173 的页面执行）
// 场景一：简单请求，只读被拦截
fetch('http://localhost:8080/api', { method: 'GET' })
  // 删除服务端 Allow-Origin 头后：Network 面板中该请求状态为 200，响应体可见，
  // 但控制台抛 "Failed to fetch" TypeError，即响应到达了浏览器但被拒绝移交 JS。

// 场景二：非简单请求，触发预检
fetch('http://localhost:8080/api', {
  method: 'PUT',
  headers: { 'Content-Type': 'application/json' }  // 非简单 Content-Type 触发 preflight
});
// Network 面板顺序：先出现一个 OPTIONS，收到 204 后浏览器才发出真实 PUT。
// 若在服务端注释掉 OPTIONS 分支的 Allow-Methods 头，则只出现 OPTIONS，PUT 永远不出网。

上述两端代码无任何业务逻辑，纯粹演示浏览器 CORS 裁决的完整链路：简单请求是"算后账"（先执行后拦截），预检请求是"先请示后执行"（先协商后放行）。
```

### 4. 常见误区与进阶思考
误区一：把 CORS 当作服务端安全机制或 API 防火墙。CORS 是协作式策略——服务端只是"声明"许可，执行裁决的是浏览器。攻击者使用 curl、代理、爬虫、服务端脚本访问同一接口时，CORS 头完全不产生任何阻止效力。因此 CORS 绝不能替代身份认证、授权与防重放机制；反过来，任何依赖"跨源浏览器请求无法访问"来保护数据的架构设计都是纸墙。同时要理解 Credentials 模式：若请求携带 cookie（fetch 的 credentials:'include'），服务端 Access-Control-Allow-Origin 不能使用通配符 *，必须回显精确源，且需额外配合 Access-Control-Allow-Credentials: true——这进一步证明该协议是"浏览器与页面源"之间的一对一授权对话，而非资源级别的开放授权。

误区二：认为"跨域拦截 = 请求没发出去"。两种拦截语义截然不同：预检失败发生在发送前，真实请求不出网，服务端无任何副作用；简单请求的拦截发生在响应读取后，请求早已到达服务端，副作用（写库、触发消息、扣减积分）已真实发生，JS 只是拿不到响应体。简单请求 + 副作用接口 = 天然的 CSRF 攻击面，因为浏览器会自动携带目标站点的 cookie，且 HTML 表单就能伪造这类请求；CORS 对读的封锁并不能阻止写副作用的执行。CSRF 必须靠 SameSite Cookie、CSRF token、自定义头配合预检等手段解决，CORS 本身不是 CSRF 的修复方案。

进阶思考题：为什么 GET 请求一旦携带一个自定义头（如 X-Requested-With）就会从"简单请求"升级为"预检请求"？请从"HTML 表单能否伪造该请求"这个角度推演预检机制的防御对象到底是什么——想明白这个问题，就真正理解了简单请求白名单（有限的请求方法、有限的 Content-Type、禁止自定义头）并非性能优化，而是一条精心划定的安全边界。答案方向：表单只能发起 GET/POST 且 Content-Type 受限、无法携带自定义头；预检的本质是让服务端对"非浏览器原生页面可伪造、必须由真实脚本才能构造"的请求显式表态，从而把脚本请求与页面导航/表单提交两种信任模型区分开。
