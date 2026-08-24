---
title: "每日基础技术总结 · 2026-08-25 · DNS 解析流程"
date: 2026-08-25 06:56:15
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-25 · DNS 解析流程

## 📚 今日主题

> **DNS 解析流程**（前端底层与计算机基础）

### 1. 核心概念速览
DNS（Domain Name System）是一个分布式的、层次化的键值数据库，核心功能是将人类可读的域名（如 example.com）解析为机器可读的资源记录（如 A/AAAA 记录中的 IP 地址），以及处理 CNAME、MX、TXT 等各类记录。它的本质是一个基于 UDP/TCP 53 端口的应用层协议，底层依赖 DNS 报文格式与分布式命名树。DNS 解决的核心问题是：在无中心权威、海量域名、频繁变化的网络环境中，如何高效、可靠、可扩展地完成名字到资源的映射。该机制采用『树形命名空间 + 委派授权 + 逐级缓存』的设计，根服务器只负责顶级域（如 .com）的 NS 指向，顶级域只负责二级域，以此类推，最终由权威服务器返回精确答案。在整个计算机体系中，DNS 是互联网的『目录服务』，是几乎所有网络请求（HTTP、WebSocket、邮件）的第一步；专业工程师必须掌握它，因为任何一次线上故障、域名劫持、CDN 调度、微服务服务发现都绕不开 DNS 的底层行为。

### 2. 底层原理剖析
DNS 解析的底层机制由两个关键维度组成：查询模式（递归/迭代）与缓存体系（TTL）。递归查询发生在客户端与本地解析器（如系统配置的 8.8.8.8 或 114.114.114.114）之间：客户端只发起一次请求，解析器负责完成全部查找并返回最终答案。迭代查询发生在解析器与各级 DNS 服务器之间：解析器依次向根服务器、顶级域服务器、权威服务器发起查询，每一步只获得一个『下一级服务器的地址』，最终拿到记录。具体流程可描述为：① 客户端检查本地 DNS 缓存与 hosts 文件；② 未命中则向配置的本地解析器发起递归请求；③ 本地解析器若缓存未命中，则向根服务器查询顶级域 NS；④ 根服务器返回顶级域（如 .com）的 NS 与 glue record；⑤ 解析器向 .com 顶级域服务器查询 example.com 的 NS；⑥ 顶级域返回 example.com 权威服务器地址；⑦ 解析器向 example.com 权威服务器查询完整记录；⑧ 权威服务器返回 A/AAAA/CNAME 等资源记录；⑨ 解析器将结果按 TTL 缓存并返回给客户端。整个过程中，每一级都可能出现 CNAME 别名，需要继续查询目标域名。TTL 是缓存的生命周期，决定任何一层可以复用结果多久，负缓存（NXDOMAIN）也有 TTL。与前端已有概念的对比：DNS 的『递归/迭代』类似于前端事件传播中的『冒泡/捕获』——同样是多级链路中的路径选择，但 DNS 更强调请求的委派与聚合；DNS 的分层命名空间类似于前端模块解析中的 webpack resolve，从根目录/node_modules 逐级向上查找，但 DNS 的各级服务器是分布式自治的，而前端模块解析是本地文件系统的确定性查找；DNS 的 TTL 缓存与浏览器 HTTP 缓存的 max-age 本质同构，都是通过时间戳避免重复请求，但 DNS 缓存发生在系统级和网络级，对应用透明。

### 3. 基础代码与实战验证
```text
以下为文字化伪代码，描述一个迭代解析器的核心逻辑，用于验证 DNS 的分层查找与 CNAME 处理。

function resolve(domain, qtype='A'):
    cache = load_local_cache()
    if cache.has(domain) and not expired(cache[domain]):
        return cache[domain]           # 命中缓存，跳过网络请求

    ns_list = ROOT_HINTS                # 内置 13 个根服务器 IP
    current_name = domain

    loop:
        response = udp_query(ns_list, current_name, qtype)   # 向当前层服务器发送 DNS 查询
        if response.rcode == NXDOMAIN:
            raise DomainNotExist        # 权威判定域名不存在，可负缓存
        if response.rcode == NOERROR and response.answer:
            answer = response.answer[0]
            if answer.type == qtype:
                cache.store(domain, answer.rdata, ttl=answer.ttl)   # 按 TTL 写入缓存
                return answer.rdata
            elif answer.type == CNAME:
                current_name = answer.rdata   # 别名替换，继续查询新目标
                continue
        # 无 answer 时，response.authority 包含下一级 NS 记录
        if response.authority:
            next_ns = get_next_ns(response.authority)      # 从 authority 提取 NS 名称
            ns_list = resolve_glue_address(next_ns, response.additional)  # 优先用 additional 中的 glue IP
            continue
        else:
            raise ResolutionFailed

说明：udp_query 负责构造 DNS 报文（Header 中的 ID、Flags、QDCOUNT 等），并解析响应。实际解析器还须处理超时重传、EDNS、DNSSEC、TCP fallback（响应超过 512 字节时）。这段伪代码反映了迭代查询的本质：每一步只获取下一级服务器的位置，直到权威服务器返回最终记录。
```

### 4. 常见误区与进阶思考
误区一：认为『DNS 就是查 A 记录』。实际解析流程中会大量遇到 CNAME、MX、NS、TXT、AAAA 等记录，而且 CNAME 会导致额外的查询链；专业工程师若只关注 A 记录，会在配置 CDN、邮件服务或排查延迟时遗漏关键环节。误区二：认为『每次解析都要从根服务器开始』。实际上各级缓存（浏览器缓存、OS 缓存、本地解析器缓存、ISP 缓存）会大幅减少请求路径；若忽略 TTL 和缓存，会错误估计 DNS 请求量，也可能在变更域名记录后因缓存未过期而看不到生效。

思考题：假设一个域名的权威 NS 记录 TTL 设为 0，但本地递归解析器出于性能考虑强制缓存该记录 5 分钟。此时域名所有者将 NS 指向另一组服务器，为什么新记录可能 5 分钟后才生效？请从 DNS 缓存层级的『父区缓存子区委派信息』这一机制解释，并说明这暴露了 DNS 最终一致性的哪一面。
