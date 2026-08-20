---
title: "每日基础技术总结 · 2026-08-04 · 渲染阻塞资源与 defer/async 脚本执行时机"
date: 2026-08-04 08:00:00
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-08-04 · 渲染阻塞资源与 defer/async 脚本执行时机

## 📚 今日主题

> **渲染阻塞资源与 defer/async 脚本执行时机**（前端底层与计算机基础）

### 1. 核心概念速览
渲染阻塞资源是指浏览器在构建关键渲染路径时，必须等待其加载并解析完成才能继续后续步骤的资源。核心包括 CSS（构建 CSSOM 的前提）和同步 JavaScript（可能修改 DOM/CSSOM，因此解析器必须暂停等待其执行完毕）。机制：HTML 解析器逐字解析字节流，遇到同步脚本时立即停止解析并执行；遇到样式表时虽然不阻塞 HTML 解析，但阻塞渲染树构建。defer 与 async 是 script 标签的属性，用于改变脚本的加载与执行时序：defer 使脚本在文档解析完成后、DOMContentLoaded 触发前按文档顺序执行；async 使脚本在下载完成后立即执行，不保证顺序，且可能阻塞解析。该知识点属于浏览器渲染引擎与 Web 性能领域，是理解关键渲染路径、首屏性能优化的基础。专业工程师必须掌握，因为它直接决定了资源的加载策略，影响 LCP、FCP 等核心指标，是性能调优不可回避的底层机制。

### 2. 底层原理剖析
底层运行机制可描述为以下循环：

parse(document):
  while (token = nextToken()):
    if token is script:
      if script has src:
        if async: start download; continue parse; on download complete: execute now (pause parse)
        elif defer: start download; continue parse; after parse complete: execute in order before DOMContentLoaded
        else: download and execute immediately, pausing parse
      else: execute inline immediately, pausing parse
    elif token is style:
      start download; continue parse; but rendering waits for CSSOM

CSS 阻塞的本质：浏览器必须构建 CSSOM 才能生成渲染树。遇到 <link rel='stylesheet'> 时，下载和解析是异步的，但渲染流程会等待 CSSOM 完成。同时，后续脚本执行前必须等待前置样式表解析完毕，因为脚本可能查询计算样式。

defer/async 的异同：
- 相同：都不阻塞 HTML 解析的下载过程。
- 不同：执行时机不同。defer 保证在解析完成后、DOMContentLoaded 前按顺序执行；async 下载完成立即执行，时机不可控，可能阻塞解析。

这与前端工程师熟悉的 Java 接口与 TS 接口的区别类似：都是对同一个概念（接口/脚本执行）在不同约束条件下的不同语义。Java 接口是显式的契约，必须在类声明中 implements，编译期强约束；TS 接口是结构性的，只要形状匹配即可，运行期无影响。普通脚本、defer、async 三者的差异不在资源加载方式，而在于脚本执行与文档解析的耦合强度：普通脚本强耦合（同步阻塞），defer 弱耦合（延迟到解析后），async 无耦合（独立执行）。这正如 Java 接口和 TS 接口，一个强调时序上的显式绑定，一个强调结构上的自由匹配。

### 3. 基础代码与实战验证
```text
验证代码（无框架，直接浏览器控制台观察）：

<link rel='stylesheet' href='/slow.css'> <!-- 服务端延迟 3 秒返回 -->
<script>console.log('inline')</script>
<script defer src='defer.js'></script>
<script async src='async.js'></script>

<!-- body 末尾 -->
<script>console.log('body-end')</script>
<script>
  document.addEventListener('DOMContentLoaded', () => console.log('DOMContentLoaded'));
  window.addEventListener('load', () => console.log('load'));
</script>

defer.js 内容：console.log('defer')
async.js 内容：console.log('async')

关键行注释：
- /slow.css 通过服务端延迟 3 秒返回。解析器遇到 <link> 会发起请求，但继续解析。
- 遇到内联脚本时，解析器暂停。由于前面有未完成的样式表，且脚本可能依赖样式，所以解析器会等待样式表加载并构建 CSSOM 后才执行脚本。这证明 CSS 不仅阻塞渲染，也阻塞后续脚本。因此 'inline' 会在 CSS 加载完成后输出。
- 遇到 defer 脚本，浏览器并行下载，但不在此时执行。文档解析完成后（即所有同步脚本执行完），defer 脚本按文档顺序执行，然后触发 DOMContentLoaded。所以 'defer' 会在 'body-end' 之后、'DOMContentLoaded' 之前输出。
- 遇到 async 脚本，浏览器并行下载。一旦下载完成，立即在空闲或当前解析点执行，因此 'async' 的输出时机可能在 'inline' 之后、'defer' 之前或之后，完全取决于下载速度。这验证了 async 不保证顺序，且执行时可能阻塞解析器。

实际输出顺序可能因网络而异，但核心规律可验证：CSS 阻塞内联脚本；defer 延迟到解析完成后；async 下载完成即执行，顺序不可控。
```

### 4. 常见误区与进阶思考
常见误区：
1. 认为 async 和 defer 都是“异步”所以不阻塞解析。事实上，两者只是不阻塞资源下载，但 async 脚本一旦下载完成就会立即执行，执行时必然占用主线程并暂停解析器（如果解析尚未完成）。defer 脚本的执行阶段在解析完成后，虽然不阻塞解析，但会阻塞 DOMContentLoaded 触发。因此，“异步”仅指下载阶段，执行阶段仍可能阻塞关键路径。
2. 认为 CSS 只阻塞渲染，不影响脚本。实际上，在脚本执行之前，如果存在尚未加载完成的样式表，脚本必须等待 CSSOM 构建完成，因为脚本可能读取如 getComputedStyle 等依赖样式的 API。因此，CSS 会阻塞后续所有同步脚本的执行。这是很多性能优化文档未强调的细节。

进阶思考题：假设一个页面包含：
- 一个内联脚本 A（位于 head 中）
- 一个 async 脚本 B（引用外部文件，下载需要 100ms）
- 一个 defer 脚本 C（引用外部文件，下载需要 1ms）
- 一个内联脚本 D（位于 body 末尾）
请问在 A 执行时，B 和 C 的下载状态如何？当 D 执行时，B 和 C 是否已经开始下载？在 DOMContentLoaded 触发时，B 和 C 哪个一定已经执行？哪个可能未执行？为什么？
