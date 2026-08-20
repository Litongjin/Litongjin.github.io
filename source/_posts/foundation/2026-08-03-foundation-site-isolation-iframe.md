---
title: "每日基础技术总结 · 2026-08-03 · 浏览器站点隔离（Site Isolation）与 iframe 进程边界"
date: 2026-08-03 08:00:00
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-08-03 · 浏览器站点隔离（Site Isolation）与 iframe 进程边界

## 📚 今日主题

> **浏览器站点隔离（Site Isolation）与 iframe 进程边界**（前端底层与计算机基础）

### 1. 核心概念速览
站点隔离（Site Isolation）是Chromium多进程架构下的一项安全机制，其本质是将“站点”（scheme + eTLD+1）作为进程分配的信任边界：同一站点的文档被放置在同一个渲染进程中，不同站点的文档（包括跨站iframe）强制使用不同的渲染进程，从而在操作系统层面获得物理内存隔离。
它解决的核心问题是：同源策略（SOP）只约束脚本对跨源资源的API访问，但无法防御CPU侧信道攻击（如Spectre）导致的跨站数据读取。站点隔离通过进程的独立地址空间，使攻击者无法在自身进程内直接读取其他站点的内存。
机制上，浏览器在导航和iframe创建时，根据目标URL计算site，并通过进程表（process model）决定复用或创建渲染进程；跨站iframe会升级为Out-of-Process iframe（OOPIF），其生命周期完全独立于父页面进程。
在整个计算机体系中，站点隔离是浏览器安全模型对操作系统进程抽象的深度运用，也是现代浏览器纵深防御（defense-in-depth）的基石。专业工程师必须掌握，因为进程边界直接影响页面渲染性能、iframe行为、内存占用、调试方式，也决定了应用在极端安全场景下的可靠性。

### 2. 底层原理剖析
底层原理涉及三个层面：site的定义、进程分配决策、iframe的OOPIF化。
首先，site不是origin。origin是scheme+host+port，用于数据访问控制；site是scheme+registrable domain（eTLD+1），用于进程隔离。例如，https://a.example.com:8443与https://b.example.com的origin不同，但site都是https://example.com，因此可以共享进程；而http://example.com与https://example.com的site不同，必须分进程。
其次，导航时的进程选择伪代码如下：
    def choose_process_for_navigation(url, current_process):
        target_site = compute_site(url)   # 从URL提取scheme和eTLD+1
        if current_process is None:
            return get_or_create_process(target_site)
        if current_process.site == target_site:
            return current_process
        else:
            return get_or_create_process(target_site)
对于iframe，当HTML解析器遇到<iframe>标签且src的site与父页面不同，渲染进程不会直接渲染该iframe，而是向浏览器进程发起创建OOPIF的请求；浏览器进程分配新渲染进程，并建立代理（proxy）通道。此后，父页面与iframe之间的一切交互（如postMessage、focus、render指令）都经过浏览器进程转发，通过IPC消息序列化。
这一模型与Java接口、TS接口的对比：Java接口是编译期方法签名的显式契约，TS接口是结构类型系统下的兼容性检查，二者都是语言层面的静态抽象，不改变运行时内存布局；而站点隔离是浏览器运行时强制执行的进程权限边界，属于操作系统级隔离。可以类比：同源策略像是“接口”——规定了你能调用什么；站点隔离像是“内存保护”——决定了你能看见什么。前端的模块作用域或CSS作用域是逻辑边界，而进程边界是物理边界，无法被JS绕过。

### 3. 基础代码与实战验证
```text
实际验证最直接的方式是利用Chrome的进程模型观察工具。以下是一段极简页面：

    <!DOCTYPE html>
    <html>
    <head><title>OOPIF验证</title></head>
    <body>
      <!-- 同站iframe（假设本页部署在https://example.com）与父页面同进程 -->
      <iframe id="same" src="/inline.html"></iframe>
      <!-- 跨站iframe，其site与父页面不同，将被放入独立渲染进程 -->
      <iframe id="cross" src="https://example.net"></iframe>
      <script>
        const crossFrame = document.getElementById('cross');
        // 尝试直接读取跨站frame的document，会被同源策略拒绝。
        // 即使没有进程隔离也会拒绝，但在这里，进程隔离进一步保证
        // 该frame的整个内存环境不在当前进程中。
        try {
          crossFrame.contentDocument; // 触发SecurityError
        } catch (e) {
          console.log('跨站DOM访问被拒绝:', e.name);
        }
        // postMessage是OOPIF与父页面之间唯一的异步通信方式，
        // 消息会被序列化并通过IPC传递给独立进程中的iframe。
        crossFrame.contentWindow.postMessage({type:'ping'}, 'https://example.net');
        // 在Chrome任务管理器（Shift+Esc）中可以看到两个渲染进程：
        // 一个包含父页面+同站iframe，另一个仅包含跨站iframe。
        // 也可在chrome://process-internals中查看进程ID和站点归属。
      </script>
    </body>
    </html>

注释说明：同站iframe与父页面同进程，因为site一致；跨站iframe触发OOPIF，其进程ID与父页面不同，且无法通过任何JS API获取对方进程内的数据。该验证不需要任何框架，在本地HTTP服务器上运行即可。
```

### 4. 常见误区与进阶思考
误区一：认为同源策略（SOP）足以保障安全，站点隔离只是性能优化。事实是，SOP只限制脚本对跨源DOM和网络的访问，但无法防御微架构侧信道攻击（如Spectre）。Spectre利用CPU预测执行和缓存时间测量，可以在同一地址空间内读取任意数据；若不进行进程隔离，恶意网站可以从同一进程内的其他站点数据中泄露信息。站点隔离正是为此而设计的“硬隔离”。
误区二：混淆“站点”（site）与“源”（origin），误以为只要host不同就必然分进程。实际上，site忽略端口，并基于公共后缀列表（PSL）计算可注册域。例如https://a.example.com:8080和https://b.example.com的site相同（https://example.com），它们可以共享进程；而https://example.com和http://example.com的scheme不同，site不同，必须分进程。另外，同站跨源（如https://example.com嵌入https://sub.example.com）不会触发OOPIF，因为它们site相同，这正体现了进程隔离粒度的权衡。
深度思考题：为什么站点隔离选择“站点”（scheme+eTLD+1）作为进程边界，而不是更细粒度的“源”（scheme+host+port）？请从攻击面收缩、进程数量上限、兼容性（如通过window.open的DOM引用）和工程实现复杂度四个维度分析。
