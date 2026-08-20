---
title: "每日基础技术总结 · 2026-06-17 · CSP 的 nonce 与 hash 及 strict-dynamic 指令"
date: 2026-06-17 08:00:00
categories: [技术分享]
tags: ["技术分享", "安全基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-06-17 · CSP 的 nonce 与 hash 及 strict-dynamic 指令

## 📚 今日主题

> **CSP 的 nonce 与 hash 及 strict-dynamic 指令**（安全基础）

### 1. 核心概念速览
CSP 的 nonce 与 hash 是两种用于在 Content Security Policy 中白名单化内联脚本/样式的机制。nonce 是一次性随机数，由服务端在响应中动态生成，并注入到允许执行的 script/style 标签的 nonce 属性中；浏览器在解析时将标签上的 nonce 值同策略中声明的 nonce 值进行常量时间比较，相等则放行。hash 是将内联内容的完整字节做 SHA-256/384/512 哈希，并将计算结果以 sha256-... 形式放入策略；浏览器对实际内容计算哈希并与策略比对，一致则放行。strict-dynamic 指令改变信任模型：当一个脚本因 nonce 或 hash 被允许执行后，它通过 DOM API（如 appendChild）创建的任何新脚本都会自动继承信任，无需额外 nonce。该机制位于 HTTP 响应头与浏览器安全模型之间，解决的核心问题是在不启用 unsafe-inline 的前提下安全执行合法内联脚本，同时阻止攻击者注入任意代码。专业工程师必须掌握，因为现代前端工程大量依赖动态脚本加载与内联状态，错误配置会导致线上功能崩溃或安全防线失效。

### 2. 底层原理剖析
底层原理剖析：浏览器在解析 HTML 时，对每个 script 元素在执行前执行如下判定（伪代码）：

function isScriptAllowed(el, policy) {
  if (el.trustedByStrictDynamic) return true;
  if (policy.scriptSrc.contains('strict-dynamic')) {
    if (el.nonce && policy.nonces.contains(el.nonce)) return true;
    if (el.textContent && policy.hashes.contains(hash(el.textContent))) return true;
    return false;
  } else {
    // 传统逻辑：检查 nonce、hash、源白名单、unsafe-inline
  }
}

trustedByStrictDynamic 是一个内部标记，当脚本 A 通过 DOM API 创建 script B 时，如果 A 已通过 CSP 校验，则 B 被标记为 trustedByStrictDynamic。

nonce 与 hash 的本质区别：nonce 验证的是信任传递中的瞬时凭证，需要服务端每次响应生成新值，防止重放；hash 验证的是内容指纹，适合内容固定且不需要动态变化的内联脚本。strict-dynamic 则是一种信任链模型，将信任判断从静态资源属性扩展到动态执行上下文。

与前端已有概念的异同：
- 与 SRI 对比：SRI 用 hash 校验外部资源完整性，防止 CDN 文件被篡改；CSP hash 校验内联脚本是否属于白名单，决定是否允许执行。两者都基于哈希，但 SRI 是资源加载后的完整性确认，CSP hash 是资源执行前的授权判定。
- 与 CSRF token 对比：CSRF token 是服务端签发、客户端请求时携带的防跨站请求伪造凭证；CSP nonce 是服务端签发、浏览器在解析 HTML 时使用的防未授权脚本执行凭证。两者都依赖不可预测性和服务端签发，但 CSRF token 由 JS 读取并附加到请求，CSP nonce 由浏览器内部读取并直接比对，脚本自身无法访问。
- 与前端框架的 nonce 透传对比：React/Vue 支持在渲染时给 script 标签传 nonce 属性，但这仅仅是属性值的输出，CSP 的校验发生在浏览器原生层，框架无法绕过或改变判定逻辑。

### 3. 基础代码与实战验证
```text
const http = require('http');
const crypto = require('crypto');

const inlineScript = `console.log('hash-ok')`; // 定义待哈希的内联脚本内容，浏览器会计算其 SHA-256 与策略比对
const hash = crypto.createHash('sha256').update(inlineScript).digest('base64'); // 计算 base64 编码的 SHA-256 哈希，用于拼入 CSP
const nonce = 'secretNonce'; // 演示用固定 nonce，生产环境必须每次请求由 CSPRNG 生成
const csp = `script-src 'nonce-${nonce}' 'sha256-${hash}' 'strict-dynamic'`; // CSP 头：nonce 白名单、hash 白名单、strict-dynamic

const html = `<!DOCTYPE html><html><head><meta charset='utf-8'><title>CSP Test</title></head><body>
<script nonce='${nonce}'>console.log('nonce-ok')</script> <!-- 该脚本因 nonce 匹配被允许 -->
<script>${inlineScript}</script> <!-- 该脚本无 nonce，但内容哈希匹配，因此被允许 -->
<script nonce='${nonce}'>
  var s = document.createElement('script');
  s.textContent = 'console.log(1)'; // 动态创建的内联脚本，无 nonce 且 hash 不匹配
  document.body.appendChild(s); // 在 strict-dynamic 下，因父脚本已受信任，此脚本继承信任
</script>
</body></html>`;

http.createServer((req, res) => {
  res.setHeader('Content-Security-Policy', csp);
  res.setHeader('Content-Type', 'text/html; charset=utf-8');
  res.end(html);
}).listen(8080);
```

### 4. 常见误区与进阶思考
常见误区：
1. nonce 静态化：将 nonce 硬编码在前端代码或构建产物中，导致攻击者可以提前获取并注入。正确做法是服务端每次响应动态生成不可预测的 nonce，并通过模板注入到 HTML 和 CSP 头中。
2. 误以为 strict-dynamic 可以单独使用或与 unsafe-inline 共存。实际上，strict-dynamic 必须依赖至少一个 nonce 或 hash 作为信任根；且当 strict-dynamic 存在时，浏览器会忽略 script-src 中的 unsafe-inline 以及所有源白名单（如 'self'），只信任由已信任脚本创建的脚本。这要求所有动态脚本必须由合法脚本触发，否则会被阻止。

深度思考题：
如果 CSP 策略为 `script-src 'nonce-abc' 'strict-dynamic'`，页面中有一个带 nonce='abc' 的脚本 A，A 通过 `document.createElement('script')` 创建了一个外部脚本 B（src 指向第三方域），B 被加载后尝试调用 `eval('1')`。请分析 B 调用 eval 是否成功，并说明底层判定逻辑。
