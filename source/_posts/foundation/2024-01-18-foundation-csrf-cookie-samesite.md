---
title: "每日基础技术总结 · 2024-01-18 · CSRF 攻击与同源 Cookie/Samesite 防护"
date: 2024-01-18 20:00:00
categories: [技术分享]
tags: ["技术分享", "安全进阶（Web / 认证授权 / 密码学）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-01-18 · CSRF 攻击与同源 Cookie/Samesite 防护

## 📚 今日主题

> **CSRF 攻击与同源 Cookie/Samesite 防护**（安全进阶（Web / 认证授权 / 密码学））

### 1. 核心概念速览
CSRF（跨站请求伪造）是一种攻击者利用受害者已认证的会话状态，诱导浏览器向目标资源服务器发送非用户本意请求的机制。其本质并非窃取数据，而是利用 HTTP 协议中浏览器自动携带凭证（Cookie）的特性，在目标域下执行具有副作用的操作。同源策略（SOP）仅限制脚本读取响应内容，但不阻止浏览器自动发送带 Cookie 的请求，因此 SOP 无法防御 CSRF。掌握此知识点对于构建后端安全架构至关重要，因为认证状态的管理必须区分‘身份验证’（Authentication）与‘授权/操作完整性’（Authorization/Integrity），理解这一区别是防止状态被恶意滥用的基础。

### 2. 底层原理剖析
CSRF 攻击的核心机制依赖于三个条件：1. 用户对站点拥有可信身份认证（如登录态 Cookie）；2. 目标站点使用 Cookie 等凭据进行会话识别且无额外校验；3. 用户当前保持登录状态并访问恶意页面。当恶意页面包含指向目标站的请求（无论 GET 或 POST）时，浏览器会自动附上目标域的 Cookie，服务器误认为请求合法。

与前端 CORS（跨域资源共享）对比：CORS 是浏览器基于‘跨域读’场景设计的响应头机制，通过 Access-Control-Allow-Origin 控制 JS 能否读取跨域响应；而 CSRF 针对的是‘跨域写’场景下的请求发起行为，浏览器出于协议兼容性考虑，默认允许跨域发送 GET/POST 请求并携带 Cookie。SameSite Cookie 属性从服务端角度修正了这一默认行为，将其设为 Strict 或 Lax 可禁止跨站上下文中自动携带 Cookie，从而在根源上阻断 CSRF，无需依赖前端 Token 注入逻辑。

### 3. 基础代码与实战验证
```text
// 后端 Set-Cookie 设置演示 SameSite 防护机制
// 关键参数解释：
// SameSite=Strict: 任何跨站请求都不携带 Cookie（最高安全等级）
// SameSite=Lax:   仅在顶级导航（如地址栏输入URL跳转）时携带，禁止 POST 等跨站请求携带
// SameSite=None:  必须配合 Secure=true 使用，允许跨站携带，此时需手动实现 CSRF Token 校验

res.setHeader('Set-Cookie', 'session_id=abc123; HttpOnly; Secure; SameSite=Lax');

// 若未使用 SameSite，后端必须在 Form 提交或自定义 Header 中校验 CSRF Token
// 伪代码逻辑：
// 1. 生成随机高熵字符串 token = randomBytes(32)
// 2. 将 token 存入 Session 或 Redis，并返回给前端渲染到表单隐藏域或 Meta 标签
// 3. 前端在每次写请求（POST/PUT/DELETE）中将 token 放入 X-CSRF-Token Header
// 4. 后端校验 Request.Header['X-CSRF-Token'] === Session[userId]['csrf_token']
```

### 4. 常见误区与进阶思考
误区一：认为配置了 CORS 就能防御 CSRF。CORS 解决的是 XSS 后恶意脚本读取敏感数据的问题，而 CSRF 是利用合法脚本（甚至图片、表单）发起请求，CORS 对此无能为力，除非同时限制了凭证发送。
误区二：过度依赖前端存储 CSRF Token 而忽视 SameSite。在现代浏览器广泛支持 SameSite=Lax 默认值的背景下，单纯依靠前端生成和传递 Token 增加了复杂度，应优先通过 SameSite 减少攻击面，Token 作为纵深防御手段。
思考题：如果攻击者构造一个包含 `<img src="https://bank.com/transfer?to=victim">` 的图片标签，该请求是否会被浏览器自动带上银行网站的 Cookie？如果银行后端使用了 JSON 响应而非 HTML，前端 JS 能直接读取这个 img 标签加载失败或成功的细节来推断转账状态吗？这体现了哪些安全边界？
