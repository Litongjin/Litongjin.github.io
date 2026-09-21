---
title: "每日基础技术总结 · 2024-05-15 · 认证 vs 授权：Session/Cookie 与 Token"
date: 2024-05-15 20:00:00
categories: [技术分享]
tags: ["技术分享", "安全进阶（Web / 认证授权 / 密码学）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-05-15 · 认证 vs 授权：Session/Cookie 与 Token

## 📚 今日主题

> **认证 vs 授权：Session/Cookie 与 Token**（安全进阶（Web / 认证授权 / 密码学））

### 1. 核心概念速览
认证（Authentication）与授权（Authorization）是身份鉴别与权限控制的两个正交阶段。Cookie/Session 机制基于服务端状态存储，浏览器自动携带 Cookie，服务端通过 Session ID 查找内存或持久化存储中的用户上下文；Token（如 JWT）机制基于客户端自包含凭证，每次请求需显式携带（通常在 Authorization Header），服务端通过验证签名或查询黑名单确认有效性。在计算机安全体系中，前者依赖信任的会话保持通道，后者依赖密码学或分布式一致性保障。专业工程师必须掌握二者以应对无状态扩展、跨域协作及安全合规需求。

本质差异：Cookie/Session 是将状态保存在服务端（Server-side State），Token 是将状态编码并分散到客户端（Client-side Self-contained）。

### 2. 底层原理剖析
1. Session/Cookie 流程：
   - 登录：客户端提交凭证 -> 服务端验证 -> 生成唯一 Session ID -> 写入存储（Redis/DB）-> 返回 Set-Cookie 头。后续 HTTP 请求由 UA 自动附加 Cookie 头。
   - 鉴权：服务端解析 Cookie 中的 Session ID -> 查存储获取状态对象 -> 验证业务逻辑。
   - 缺陷：强依赖服务端的存储可用性，水平扩展需共享 Session 存储或 Sticky Session；存在 CSRF 风险。

2. Token (JWT) 流程：
   - 签发：服务端用私钥对 Header.Payload.Signature 进行加密签名 -> 返回给客户端。
   - 携带：客户端将 Token 放入 Authorization: Bearer <token> 头。
   - 验证：服务端使用公钥或共享密钥验签（Verify Signature），解码 Payload 获取 claims（身份信息），无需查库即可确认身份合法性。
   - 对比前端接口概念：TS 的 Interface 是编译时的静态类型契约，用于约束数据结构形状，不存在运行时开销；Java 的 Interface 是运行时的多态契约。而 Cookie 是传输层协议约定的键值对字段，Token 是应用层自定义的加密数据块，二者皆服务于数据传输语义，但 Session 更偏向‘引用指针’（指向服务端资源），Token 更像‘数字印章’（自证清白）。

核心区别表：
| 特性 | Session/Cookie | Token (JWT) |
|---|---|---|
| 状态存储位置 | 服务端 | 客户端 (Header) |
| 服务器压力 | 高 (需查库/缓存) | 低 (仅验签) |
| 跨域支持 | 原生支持 (同源策略下) | 易实现 (CORS) |
| 撤销机制 | 即时有效 (删除记录) | 延迟有效 (需黑名单或短过期) |
| 安全性 | 依赖 HttpOnly/Secure 防窃取 | 防篡改 (签名)，但内容可能泄露敏感信息 |

伪代码流：
// Session
if (request.hasCookie('session_id')) {
   user = db.findSession(request.cookie('session_id')); // 核心IO操作
   if (!user) return 401;
}

// Token
if (request.header('Authorization').startsWith('Bearer ')) {
   token = parse(request.header('Authorization'));
   isValid = crypto.verify(token.signature, token.payload, publicKey); // 纯CPU计算
   if (!isValid) return 401;
}
