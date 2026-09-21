---
title: "每日基础技术总结 · 2025-10-10 · 浏览器安全策略 SameSite Cookie 属性对第三方跨域请求的影响"
date: 2025-10-10 20:00:00
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2025-10-10 · 浏览器安全策略 SameSite Cookie 属性对第三方跨域请求的影响

## 📚 今日主题

> **浏览器安全策略 SameSite Cookie 属性对第三方跨域请求的影响**（前端底层与计算机基础）

### 1. 核心概念速览
SameSite 属性是 HTTP Cookie 的一个安全限制参数，用于控制浏览器在跨站点请求中是否随请求携带特定 Cookie。其本质是实施基于上下文的访问控制（Contextual Access Control），旨在缓解 CSRF（跨站请求伪造）和隐私泄露风险。该机制通过在客户端（浏览器）强制插入状态检查逻辑，改变了传统 Web 应用中无条件信任同源凭证的安全假设。在计算机体系结构中，它位于应用层协议（HTTP）与存储层（Cookie Jar）的交互界面，是现代零信任架构在 Web 端的具体体现。专业工程师必须掌握它，因为它是构建现代 API 安全和用户数据隔离的基础设施，直接决定后端服务对无状态化改造及第三方集成策略的兼容性。

### 2. 底层原理剖析
浏览器的 Cookie 发送逻辑由 SameSite 值严格驱动，判定核心在于‘导航类型’与‘源上下文匹配度’：
1. Strict (strict)：仅当请求为同一站点上下文时发送。‘同站’定义通常基于 eTLD+1（有效顶级域名加一级子域）。任何指向非当前站点来源的请求（包括 top-level navigation, cross-origin fetch, iframe, script 等），浏览器均不附加该 Cookie。
2. Lax (lax)：允许部分安全的跨站 GET 请求携带 Cookie，主要指 top-level navigation（用户点击链接跳转）。对于 POST、AJAX、Image 等非顶级导航请求，视作 Strict 处理。
3. None (none)：明确声明无论何时都发送 Cookie，但必须同时设置 Secure 标志。这是实现受控第三方共享的必要条件，且面临 Chrome 等主流浏览器即将移除默认 None 行为的趋势。

底层流程伪代码：
if (request.is_cross_origin()) {
  if (cookie.sameSite == 'Strict') return null; // 丢弃
  if (cookie.sameSite == 'Lax' && !request.is_top_level_navigation()) return null;
  if (cookie.sameSite == 'None' && cookie.secure) continue; // 继续附加
}
// ... proceed with request including cookie

对比前端概念：这与 TypeScript 中的 ‘Interface Matching’（结构类型系统）不同。TS 接口关注形状的兼容性，而 SameSite 关注的是‘执行环境’的信任边界。它类似于操作系统中的 DAC（自主访问控制），根据调用者所属的进程/线程域来决定资源可见性，而非仅仅看文件名。”

### 3. 基础代码与实战验证
```text
// Node.js Express 示例：演示设置不同 SameSite 属性的响应头行为
const express = require('express');
const app = express();

app.get('/set-cookies', (req, res) => {
  // 设置 Strict: 仅同源请求携带，彻底阻断跨站身份复用
  res.cookie('session_strict', 'val1', { sameSite: 'Strict', httpOnly: true });
  
  // 设置 Lax: 允许顶级导航携带，阻断 AJAX/Fetch 携带
  res.cookie('session_lax', 'val2', { sameSite: 'Lax', httpOnly: true });
  
  // 设置 None + Secure: 需 HTTPS，明确允许跨站携带（需注意浏览器弃用趋势）
  res.cookie('session_none', 'val3', { 
    sameSite: 'None', 
    secure: true, // 强制要求 HTTPS，否则被浏览器忽略
    httpOnly: true 
  });

  res.send({ msg: 'Cookies set' });
});

app.listen(3000);

// 验证步骤:
// 1. 从 origin-A 发起 Fetch 到 origin-B/set-cookies
// 2. 检查 origin-B 返回的 Set-Cookie 头
// 3. 观察 origin-B 后续接收到的 Request Cookie 头中各值的存在情况
// Strict: 无 cookie 发送
// Lax: 无 cookie 发送 (因为是 XHR/Fetch)
// None: 携带 session_none=v3
```
