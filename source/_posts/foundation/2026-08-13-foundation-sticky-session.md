---
title: "每日基础技术总结 · 2026-08-13 · 负载均衡中的会话保持（Sticky Session）实现方式"
date: 2026-08-13 08:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-08-13 · 负载均衡中的会话保持（Sticky Session）实现方式

## 📚 今日主题

> **负载均衡中的会话保持（Sticky Session）实现方式**（网络基础）

### 1. 核心概念速览
会话保持（Sticky Session）指负载均衡器将同一客户端会话内的全部请求定向转发至同一后端服务器实例，以维持后端应用在会话上下文中保存的状态（如登录态、购物车、临时数据）。其本质是在负载均衡层建立'会话标识 → 后端实例'的绑定关系，覆盖了无状态转发策略下的有状态场景。机制上，通过识别客户端请求中携带的会话标识（Cookie、TLS Session ID、源IP等），经由哈希或映射表，将请求固定到已建立的会话所在后端。它属于L4/L7负载均衡的附加策略，与健康检查、轮询等互补，是分布式系统扩展时保留有状态服务的关键妥协方案。专业工程师必须掌握，因为它直接关系系统扩展、容灾与数据一致性，且是理解更高级分布式会话方案（如Session Replication、分布式缓存）的基石。

### 2. 底层原理剖析
实现方式分为四类：1）源IP哈希：对客户端IP做哈希运算，结果映射到后端列表。算法简单但受NAT/代理影响，且扩缩容时哈希结果变动大（可用一致性哈希缓解）。2）Cookie粘性：负载均衡器向客户端下发或改写Set-Cookie（如JSESSIONID），之后依据请求携带的Cookie值定位后端。常见三种：a) 插入Cookie（LB生成专属Cookie并插入响应）；b) 重写Cookie（修改后端生成的会话Cookie，追加后端标识）；c) 学习Cookie（读取后端应用自身Cookie值，建立映射）。3）TLS会话ID：基于TLS握手时的Session ID/Session Ticket，在L4层实现。4）应用层会话表：LB维护全局映射表，键为会话标识，值为后端实例，需处理超时与同步。核心底层逻辑：当请求到达LB时，提取会话标识→查询映射表或计算哈希→若命中则转发至对应后端；若未命中，则按正常策略选择后端并记录绑定。需要注意，会话保持是'尽力而为'，后端宕机时LB需强制移除绑定并重新选择。对比前端概念：前端工程师熟悉的HTTP Cookie与浏览器本地状态（localStorage）都是客户端侧状态维持手段，但负载均衡的Cookie粘性是在网络基础设施层操作，对应用透明；而浏览器的Cookie存储是应用主动使用的API。另一个对比：前端SPA中的路由分发与后端负载均衡都是基于请求路径/标识进行分流，但前端路由是浏览器内部逻辑，无网络状态；后端粘性会话则是分布式系统的有状态路由决策。

### 3. 基础代码与实战验证
```text
以下为极简Node.js示例，实现基于Cookie的会话保持反向代理（仅核心逻辑）：

const http = require('http');
const crypto = require('crypto');

const backends = [
  { host: '127.0.0.1', port: 8081 },
  { host: '127.0.0.1', port: 8082 }
];

// 会话映射表：cookieValue -> backendIndex
const sessionTable = new Map();

// 计算后端索引的哈希函数（一致性哈希简化版）
function hashToBackend(key) {
  const hash = crypto.createHash('md5').update(key).digest();
  return hash.readUInt32BE(0) % backends.length;
}

const server = http.createServer((req, res) => {
  // 1. 尝试从请求头中获取LB会话Cookie（假设名为LBID）
  const cookies = req.headers.cookie || '';
  const lbCookie = cookies.match(/(?:^|; )LBID=([^;]+)/);
  let backendIdx;

  if (lbCookie) {
    // 2a. 若已有LBID，查映射表；若表无记录（例如重启或超时），按哈希重新计算
    backendIdx = sessionTable.get(lbCookie[1]);
    if (backendIdx === undefined) {
      backendIdx = hashToBackend(lbCookie[1]);
      sessionTable.set(lbCookie[1], backendIdx);
    }
  } else {
    // 2b. 无LBID，生成新会话标识，随机选择后端，并写入响应头Set-Cookie
    const newSessionId = crypto.randomBytes(16).toString('hex');
    backendIdx = Math.floor(Math.random() * backends.length);
    sessionTable.set(newSessionId, backendIdx);
    // 实际会设置Expires、HttpOnly等属性，这里省略
    res.setHeader('Set-Cookie', `LBID=${newSessionId}; Path=/`);
  }

  // 3. 将请求转发到选定的后端
  const target = backends[backendIdx];
  const proxyReq = http.request({
    host: target.host,
    port: target.port,
    path: req.url,
    method: req.method,
    headers: req.headers
  }, (proxyRes) => {
    res.writeHead(proxyRes.statusCode, proxyRes.headers);
    proxyRes.pipe(res);
  });
  req.pipe(proxyReq);
});

server.listen(80);

注释说明：该代码用Map实现会话表，键为生成的LBID，值为后端索引。首次请求时生成并下发Cookie，后续请求通过Cookie定位后端。实际生产级负载均衡器（如Nginx、HAProxy）实现更复杂，涉及超时、内存清理、后端变更等。
```

### 4. 常见误区与进阶思考
误区1：认为粘性会话能够保证高可用。实际上，若绑定后端宕机，会话仍会丢失（除非有会话复制或故障转移机制）。粘性会话只保证会话期间的一致性，不提供容灾。误区2：混淆'源IP哈希'与'会话保持'。源IP哈希是一种负载均衡算法，天然具备一定的会话保持效果，但受NAT、代理和多个客户端共享IP的影响，可能将大量客户端映射到同一后端，造成负载倾斜；且当后端扩缩容时，哈希结果变化导致会话失效。进阶思考题：假设一个电商系统使用基于Cookie的粘性会话，客户端清除了Cookie后，负载均衡器如何判断新请求是否属于原有会话？如果不能判断，那么后端会话数据（如购物车）会如何？请结合HTTP无状态性和分布式缓存设计，分析粘性会话与全局会话（如Redis集中存储）的取舍。
