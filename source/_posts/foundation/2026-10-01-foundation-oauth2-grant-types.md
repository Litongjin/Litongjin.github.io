---
title: "每日基础技术总结 · 2026-10-01 · OAuth2 四种授权模式与适用场景"
date: 2026-10-01 07:04:22
categories: [技术分享]
tags: ["技术分享", "Java 后端与 Spring 生态"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-01 · OAuth2 四种授权模式与适用场景

## 📚 今日主题

> **OAuth2 四种授权模式与适用场景**（Java 后端与 Spring 生态）

### 1. 核心概念速览
OAuth2 是一种作为 RFC 6749 定义的授权框架，解决的是授权问题（Authorization），而不是认证问题（Authentication）。它定义了客户端（Client）、资源所有者（Resource Owner）、资源服务器（Resource Server）和授权服务器（Authorization Server）四者之间的交互规则。其本质是让资源所有者在不将自己的凭据（如账号密码）直接交给第三方客户端的情况下，通过授权服务器颁发一个有时效、有范围的访问令牌（Access Token）给客户端，客户端凭此令牌访问受保护资源。四种授权模式分别针对不同信任和交互场景：授权码模式（Authorization Code）适用于有后端可安全保管客户密钥的 Web 应用，且是唯一支持刷新（Refresh Token）的完整流程；隐式模式（Implicit）面向无后端的纯浏览器应用，因令牌暴露在 URL 中且无法刷新，已被授予码+PKCE 扩展取代；密码模式（Resource Owner Password Credentials）适用于客户端与资源所有者属于同一信任域的第一方应用，本质是用户将密码委托给客户端代理；客户端凭证模式（Client Credentials）不涉及用户，使用客户端自身的密钥换取令牌，用于服务间或机器身份调用。在计算机体系中，OAuth2 处在网络安全协议栈的应用层，是微服务、开放平台、API 网关的授权基础设施。工程师必须掌握，因为授权模式选择不当会直接导致令牌泄露、权限放大或无法通过安全审计，同时也是理解 JWT、OpenID Connect、SAML 等后续安全技术的基石。

### 2. 底层原理剖析
所有模式的共同点是：授权服务器是唯一的令牌签发者，客户端必须通过 grant_type 字段声明使用的授权模式，授权服务器根据该字段选择对应的校验分支，验证通过后返回一个包含 access_token 的 JSON 载荷。四种模式的本质差异只在第一步：客户端如何从资源所有者处获得授权凭证。

授权码模式流程：
1. 资源所有者在用户代理（浏览器）中访问客户端，客户端将用户重定向到授权服务器的 /authorize 端点，带上 response_type=code、client_id、redirect_uri、scope 等参数。
2. 授权服务器通过自身认证机制确认资源所有者身份（OAuth2 不规定如何认证，通常配合 OpenID Connect 实现），并展示客户端所请求的权限范围，资源所有者同意后，授权服务器生成一个一次性授权码 code，并通过 HTTP 302 重定向到客户端预设的 redirect_uri，在 query 参数中附加 code。
3. 客户端后端收到 code 后，通过 POST 请求授权服务器的 /token 端点，以 grant_type=authorization_code + code + client_secret 交换令牌。这里的关键是 client_secret 只能出现在服务端到授权服务器的直接通道中，不能暴露给浏览器。
4. 授权服务器校验 client_id 和 client_secret 匹配，且 code 未过期、未使用、绑定的 redirect_uri 一致，则签发 access_token（以及可选的 refresh_token）。code 是一次性的，其有效期通常只有几十秒，降低截获后的攻击窗口。

隐式模式流程：
1. 同授权码模式的第一步，但 response_type=token。
2. 授权服务器直接返回 access_token 放在 URL fragment 中（如 #access_token=...），浏览器不会将 fragment 发送到服务器，但会保留在本地历史中。该模式无 client_secret 参与，无法安全地执行刷新和撤销，因此已被 PKCE 强化的授权码模式取代。PKCE（RFC 7636）的核心是在客户端动态生成一个 code_verifier，将它的哈希 code_challenge 随授权请求发送，在换取 token 时出示原始 code_verifier，授权服务器校验两者是否匹配，从而在无法安全保存 client_secret 的公共客户端上侦测授权码窃取。

密码模式流程：
1. 客户端直接通过输入框收集资源所有者的用户名密码，并向 /token 端点发送 grant_type=password 和 username、password，同时带上自己的 client_id 和 client_secret（如果有）。
2. 授权服务器验证资源所有者的凭据无误后直接返回令牌。该模式将用户凭据暴露给了客户端，因此仅适用于客户端和授权服务器属于同一信任域（如官方 App）的场景。

客户端凭证模式流程：
1. 客户端直接向 /token 端点发送 grant_type=client_credentials，并携带自己的身份凭据（client_id + client_secret）。
2. 授权服务器核对客户端身份后返回一个代表客户端自身授权的 access_token。整个流程没有任何资源所有者参与，适用于服务到服务的机器通信。

与前端已有概念的异同：OAuth2 的授权码模式与前端 fetch 跨域处理中的重定向和 postMessage 一样，都是通过间接跳转来传递信任凭证，避免直接暴露敏感信息。Token 的前端存储（localStorage 或内存）本身不是 OAuth2 关心的范围，OAuth2 只规定签发和校验协议。OAuth2 的 scope 类似于前端权限模型中的细粒度权限位（如 ACL 或 RBAC 中的 action），但 OAuth2 的 scope 由第三方定义，资源服务器负责解释。另一个易混淆点是 OAuth2 与 Cookie 机制：Cookie 是浏览器自动携带的身份状态，OAuth2 则要求客户端显式地在请求头中携带令牌，二者可以共存但语义不同。掌握 OAuth2 应理解其与 CORS 的层次差异：CORS 是浏览器同源策略的放行机制，OAuth2 是 API 访问授权机制，即使 CORS 允许跨域，没有有效的 access_token，资源服务器仍应拒绝请求。

### 3. 基础代码与实战验证
```text
以最核心的授权码模式为例，token 交换环节可用以下文本化伪代码精确重现（所有参数放在同一行时用 & 分隔）：

POST https://authorization-server.example.com/oauth2/token
Header: Content-Type: application/x-www-form-urlencoded
Header: Authorization: Basic <对 client_id:client_secret 做 Base64 编码后的字符串>   // 客户端身份凭证，仅在服务端携带
Body:
  grant_type=authorization_code   // 声明采用授权码模式，授权服务器据此走 code 校验分支
  code=<一次性授权码>              // 由第一步从重定向 URL query 中取得
  redirect_uri=<注册过的一个回调地址> // 必须与发起授权请求时完全相同，防止授权码被劫持后重放
  client_id=<客户端 ID>            // 与 Authorization 头内容一致

授权服务器接收后执行以下校验逻辑：
  1. 校验 client_id 与 client_secret 是否匹配当前注册客户端；
  2. 校验 code 在有效期内且首次使用；
  3. 校验 redirect_uri 与授权请求时一致；
  4. 校验通过后生成 access_token（和 refresh_token），返回 200 JSON。

资源访问时：
  GET /api/private
  Header: Authorization: Bearer <access_token>
  // 资源服务器无需知道用户凭据，只需验证 token 的签名或调用 introspection 接口，确认 token 有效且 scope 允许。

如果想要验证整个流程，可在本地用两个命令行 curl 模拟：
  curl -L 'https://authorization-server.example.com/authorize?client_id=...&redirect_uri=...&response_type=code'
  得到 302 和 code 后，再执行：
  curl -X POST https://authorization-server.example.com/token -H 'Authorization: Basic ...' -d 'grant_type=authorization_code&code=...&redirect_uri=...'
  观察返回的与手动解析到的 access_token 是否一致。
```

### 4. 常见误区与进阶思考
误区1：将 OAuth2 的授权码当作身份认证结果，认为拿到 access_token 就是用户已通过认证。实际上 OAuth2 只授权资源访问，不传递用户信息。身份认证需要使用 OpenID Connect 扩展，通过 ID Token 来获得用户声称。误将 OAuth2 的 access_token 用于身份声称会导致 userinfo endpoint 被错误信任或越权。

误区2：认为隐式模式仍然是可行的现代方案。实际上在公认的 OAuth 2.1 草案中，隐式模式和密码模式都被移除。隐式模式令牌暴露在 URL 中，且无法使用 refresh_token；密码模式将用户密码暴露给客户端，仅用于遗留信任场景。正确的现代方案是授权码模式 + PKCE，用于所有公共客户端。

思考题：在授权码模式中，code 为什么要绑定 redirect_uri？如果客户端在 code 换 token 时不校验 redirect_uri（或授权服务器不校验），攻击者如何利用已截获的 code 完成令牌盗取？请从 token 交换请求的参数组合角度分析阻断机制。
