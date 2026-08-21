---
title: "每日基础技术总结 · 2026-06-21 · 会话固定攻击与登录后的 Session ID 轮换"
date: 2026-06-21 08:00:00
categories: [技术分享]
tags: ["技术分享", "安全基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-06-21 · 会话固定攻击与登录后的 Session ID 轮换

## 📚 今日主题

> **会话固定攻击与登录后的 Session ID 轮换**（安全基础）

### 1. 核心概念速览
会话固定攻击（Session Fixation Attack）是一种会话劫持变体，其本质是攻击者预先向受害者注入一个已知的会话标识符（Session ID），并诱使受害者使用该标识符完成身份认证；由于服务端在认证后未更换会话 ID，攻击者凭借预先持有的 ID 即可获得受害者的已认证会话。该攻击利用的是『认证前与认证后会话标识的同一性』这一缺陷。解决机制即『登录后的 Session ID 轮换』：在用户认证成功时，服务端必须立即废弃旧会话 ID，生成新的随机 ID，并确保新 ID 与认证后的会话绑定，使攻击者预置的 ID 失效。它在整个安全体系中属于 Web 会话管理层的核心防护措施，是 OWASP Top 10 中 Broken Authentication 的典型防御点。专业工程师必须掌握，因为会话管理是前后端协作的基石，任何认证系统的漏洞都直接导致账户接管，而该防御原理是理解更高级会话保护（如双绑定、令牌轮换）的前提。

### 2. 底层原理剖析
底层机制可用以下流程精确描述：
1. 服务端为匿名用户创建会话时，生成 ID（如 `SID`），并通过 Set-Cookie 下发给客户端；此时该 ID 关联的是『未认证态』。
2. 攻击者先让受害者使用攻击者已知的 `SID_evil` 访问应用（通过 URL 注入、Cookie 注入、子域名写入等方式）。
3. 受害者使用 `SID_evil` 登录，服务端校验密码成功后，若仍以 `SID_evil` 作为会话标识并切换状态为『已认证』，则攻击者只要发送携带 `SID_evil` 的请求，服务端即识别为合法用户。
4. 正确做法：认证成功后，服务端调用 `session_regenerate_id()`（PHP）或 `request.session.clear(); request.session.save(); request.session.create()`（框架层），本质是：
   - 销毁当前会话数据对应的旧 ID 映射；
   - 生成一个高熵随机新 ID；
   - 将原会话数据（如用户主键、角色）迁移到新 ID 下；
   - 新 Set-Cookie 覆盖客户端旧 ID。
关键点：轮换必须发生在『认证成功的响应』中，且旧 ID 必须立即失效，不能保留任何可接受旧 ID 的窗口。
与前端既有概念的对比：这类似于 TypeScript 的 `interface` 与 Java 的 `interface` 的区别——前者是结构类型系统（编译期约束，运行时无实体），后者是运行时引用类型（对象可 `instanceof` 检查）。会话 ID 轮换本质上是一种『状态迁移时的身份重建』，类似前端中受控组件中 `key` 变化触发组件完全重新挂载，而不是复用旧实例。如果不更换 `key`，React 会保留子组件内部状态导致旧 UI 残留；同理，不更换 Session ID，旧身份状态被攻击者复用。另一个类比是 OAuth 2.0 的 refresh token 轮换：每次使用后必须更换新令牌，防止重放。底层原则一致：高价值凭证（已认证会话）绝不能在同一标识符上长期复用。

### 3. 基础代码与实战验证
以下为极简 Node.js HTTP 会话固定演示（不依赖框架，使用原生 crypto 与 http）：

```js
const http = require('http');
const crypto = require('crypto');

// 内存会话存储：id -> {userId, isAuthenticated}
const sessions = new Map();

// 生成新的随机 Session ID（高熵 128 位）
function generateSessionId() {
  return crypto.randomBytes(16).toString('hex');
}

function parseCookies(req) {
  const header = req.headers.cookie;
  if (!header) return {};
  return Object.fromEntries(header.split(';').map(kv => {
    const [k, ...v] = kv.trim().split('=');
    return [k, v.join('=')];
  }));
}

const server = http.createServer((req, res) => {
  const cookies = parseCookies(req);
  let sid = cookies.sid;

  // 如果没有会话 ID，则创建一个新会话（未认证）
  if (!sid || !sessions.has(sid)) {
    sid = generateSessionId();
    sessions.set(sid, { userId: null, isAuthenticated: false });
    res.setHeader('Set-Cookie', `sid=${sid}; HttpOnly; Path=/`);
  }

  const session = sessions.get(sid);

  // 登录接口：验证用户名密码（这里固定模拟成功）
  if (req.url.startsWith('/login')) {
    // 认证成功后，必须执行 Session ID 轮换
    const newSid = generateSessionId();
    const authData = { userId: 42, isAuthenticated: true };

    // 销毁旧会话，并创建新 ID 关联已认证数据
    sessions.delete(sid);          // 旧 ID 立即失效
    sessions.set(newSid, authData); // 新 ID 绑定已认证状态

    // 用新 ID 覆盖客户端 Cookie
    res.setHeader('Set-Cookie', `sid=${newSid}; HttpOnly; Path=/`);
    res.end('Logged in, SID rotated');
    return;
  }

  // 其他请求，检查是否已认证
  if (session.isAuthenticated) {
    res.end(`Hello user ${session.userId}`);
  } else {
    res.end('Not authenticated');
  }
});

server.listen(3000);
```

关键注释：
- `crypto.randomBytes(16)` 生成 128 位随机数，保证 ID 不可预测。
- `sessions.delete(sid)` 在认证成功后立刻删除旧映射，攻击者即使持有旧 ID 也会因查不到会话而失效。
- 将认证数据（`userId`）放入新 ID 对应的对象中，实现状态迁移。
- 实际生产还需设置 `Secure`、`SameSite` 等属性，但核心机制不变。

### 4. 常见误区与进阶思考
常见误区 1：认为只要登录后生成了新的 Session ID 就安全，但忽略了在登录请求的响应发出前，旧 ID 仍然有效。如果应用在处理登录逻辑时先返回了部分成功响应（如 JSON 写入后未及时设置新 Cookie），或者在同一请求中先让旧 ID 访问了敏感数据，攻击者仍可能利用旧 ID。正确的做法是：认证成功的原子操作中同时完成『删除旧会话、创建新会话、设置新 Cookie』，且整个过程中间不允许任何并发请求命中旧会话。
常见误区 2：只轮换 Session ID 但不轮换会话中的其他绑定凭证（如 CSRF Token、会话固定时攻击者注入的额外参数）。如果攻击者通过 URL 注入了一个攻击者已知的 CSRF Token，而登录后该 Token 未重新生成，则攻击者仍可配合固定会话发起 CSRF。因此 Session ID 轮换必须与认证相关令牌的重新生成协同进行。
思考题：假设一个应用在用户登录后执行了 Session ID 轮换，但攻击者在受害者登录前就已经通过某种方式在受害者浏览器中植入了恶意 Service Worker，该 Service Worker 能够拦截所有网络请求并自动附带攻击者指定的 Cookie。在这种情况下，轮换 Session ID 能否防御会话固定？如果不能，说明攻击链路中的哪个环节绕过了服务端的轮换机制？这实际上揭示了会话固定防御的一个前提假设：客户端环境完全受控于用户。请深入分析服务端与客户端信任边界的本质。
