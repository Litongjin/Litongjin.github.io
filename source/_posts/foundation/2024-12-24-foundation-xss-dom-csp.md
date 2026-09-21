---
title: "每日基础技术总结 · 2024-12-24 · XSS：存储/反射/DOM 型与 CSP 防护"
date: 2024-12-24 20:00:00
categories: [技术分享]
tags: ["技术分享", "安全进阶（Web / 认证授权 / 密码学）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-12-24 · XSS：存储/反射/DOM 型与 CSP 防护

## 📚 今日主题

> **XSS：存储/反射/DOM 型与 CSP 防护**（安全进阶（Web / 认证授权 / 密码学））

### 1. 核心概念速览
跨站脚本攻击（XSS）本质是 Web 应用对用户输入数据的信任边界失效，导致恶意脚本注入并作为可信来源在受害者浏览器上下文中执行。其核心危害并非数据窃取本身，而是劫持客户端执行上下文，获取敏感信息或发起未授权操作。分为三类：反射型（非持久化，通过 URL/参数触发）、存储型（持久化，存入数据库后对访问者生效）、DOM 型（纯客户端逻辑缺陷，不经过服务端渲染即被恶意构造修改）。CSP（内容安全策略）是通过 HTTP Header 向浏览器声明受信任的资源源（SRC），从机制上阻断非法脚本加载与执行，是纵深防御体系中最后一道关键防线。工程师必须掌握它，因为它是前后端交互中最常见的逻辑漏洞，直接关联数据完整性与客户机控制权。

### 2. 底层原理剖析
XSS 的发生依赖于三个必要条件：1. 用户输入；2. 应用程序将未经充分处理的输入嵌入到可执行上下文（HTML/JS/CSS）中；3. 浏览器将该片段解析为代码执行。底层机制在于 HTML 解析器与 JavaScript 引擎的交互边界模糊。
- 反射型：恶意 Payload 经 HTTP 请求到达服务端 -> 服务端读取请求参数 -> 服务端响应中将参数值拼接进 HTML/JS -> 浏览器接收并解析执行。本质是无状态的、一次性的注入。
- 存储型：Payload 经 HTTP 提交 -> 服务端存储至数据库/文件 -> 后续正常请求读取数据 -> 服务端将数据拼接到页面模板 -> 浏览器执行。本质是持久化的信任链污染。
- DOM 型：不涉及服务端回写。前端 JS 使用 innerHTML/document.write 等危险 API 直接操作 DOM 树时，若来源变量被篡改（如 hash 部分或 query 参数），则 DOM 树构建阶段即注入脚本。本质是客户端 JS 运行时的类型混淆与信任越界。
CSP 防护原理：浏览器在解析前检查 Content-Security-Policy Header，建立白名单。当脚本尝试加载外部资源或内联执行（<script>标签内的代码）时，若不在 allowlist 中，立即中止执行。相比传统的输入过滤（黑名单思维），CSP 是基于策略的执行隔离（白名单思维），极大缩小了攻击面。

### 3. 基础代码与实战验证
```text
// 演示 DOM-XSS 的本质：信任边界在内层
// 假设 URL: http://example.com?redirect=<script>alert(1)</script>
const urlParams = new URLSearchParams(window.location.search);
const payload = urlParams.get('redirect');

// 危险行为：直接将不可信数据写入可执行上下文（DOM 的 src 属性会触发加载执行）
// 浏览器在构建 DOM 树时，将 payload 值赋给 img 的 src 属性，解析为图片请求
// 但若 payload 包含 <img src=x onerror=alert(1)> 等，onerror 事件处理器会被注册
const img = document.createElement('img');
img.src = payload; // 此时 payload 被视为 URI 字符串，若包含协议头可能触发重定向
document.body.appendChild(img);

// 正确做法：数据只作为文本节点处理，不参与 DOM 结构构建
const safeSpan = document.createElement('span');
safeSpan.textContent = payload; // textContent 自动转义特殊字符，仅显示纯文本

// CSP 示例（HTTP Header 配置）:
// Content-Security-Policy: default-src 'self'; script-src 'self' https://cdn.example.com;
// 说明：仅允许同源和特定 CDN 的脚本，禁止所有内联脚本和非授权外部脚本执行。
```

### 4. 常见误区与进阶思考
1. 误区：认为后端做了 XSS Filter（如过滤 <script>、alert 关键字）就绝对安全。实际上，过滤逻辑常被编码变异（如大小写混合、双字节编码、Unicode 转义）绕过，且无法应对 DOM 型 XSS（因为后端根本不知道前端如何解析数据）。专业方案应是输出编码（Output Encoding）而非单纯输入过滤，并强制启用 CSP。
2. 误区：混淆 React/Vue 等现代框架的安全模型。虽然主流框架默认对插值表达式进行 HTML 转义，但若开发者显式调用 v-html、dangerouslySetInnerHTML 或将数据绑定到 href/src/on* 事件属性，仍会打开 XSS 入口。框架解决了大部分场景，但未解决全部语义上下文风险。
思考题：在零信任架构理念下，如果前端必须渲染服务端下发的不可信富文本，除了 CSP 之外，如何利用 DOMPurify 等库的沙箱机制实现‘最小权限原则’，即在允许展示基本标签（如 <strong>, <em>）的同时，剥离所有属性（包括 class, id）和事件监听器？请描述其遍历与清洗的算法逻辑。
