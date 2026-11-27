---
title: "每日基础技术总结 · 2026-11-27 · HTTP置换缓存（Swap Cache）在Nginx/OpenResty中的内存管理与淘汰策略"
date: 2026-11-27 09:50:54
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-27 · HTTP置换缓存（Swap Cache）在Nginx/OpenResty中的内存管理与淘汰策略

## 📚 今日主题

> **HTTP置换缓存（Swap Cache）在Nginx/OpenResty中的内存管理与淘汰策略**（网络基础）

### 1. 核心概念速览
Nginx/OpenResty 语境中的 HTTP 置换缓存（Swap Cache）不是一个官方内建对象，而是工程上对多级 HTTP 缓存抽象的称呼：热点响应元数据或小体留在共享内存，完整响应体落在磁盘代理缓存或外部持久层，在容量约束下按访问热度、过期时间和清理策略发生层间迁移与淘汰。它解决的是内存容量小但延迟低、磁盘容量大但延迟高的矛盾，目标是在命中率、尾延迟、I/O 放大和缓存一致性之间取得可预测平衡。其本质是分层缓存、共享内存索引、文件存储与异步清理，不是操作系统 swap，也不是将完整 HTTP body 常驻内存。它位于网络基础、反向代理、边缘加速与 AI 推理结果缓存之间；专业工程师必须掌握，因为缓存键设计、淘汰语义和 stale 策略直接决定线上命中率、内存泄漏风险、磁盘压力和故障恢复行为。

### 2. 底层原理剖析
Nginx 的 proxy_cache 是最接近该抽象的内建机制：
1 请求进入反向代理后，先由 proxy_cache_key 计算缓存键；键通常由 scheme、host、URI、查询、Vary 或业务头构成。
2 Nginx 在 keys_zone 共享内存中查找索引节点。共享内存只保存键、状态、过期时间、变体、文件路径等元数据，不保存完整响应体。
3 命中且有效时，响应体从磁盘缓存文件读取，可通过 sendfile 减少用户态复制；命中但过期时，是否返回旧值取决于 proxy_cache_use_stale、proxy_cache_background_update 等。
4 未命中时，proxy_cache_lock 可对同一键加锁，避免并发击穿；响应先写入临时文件，再原子 rename 到缓存目录，最后建立索引。
5 淘汰由 cache manager 异步执行：inactive 控制长时间未访问的条目清理；max_size 控制磁盘缓存上限，超过后按最近访问性删除文件；它不是同步 LRU，也不是内存上限。

OpenResty 的常见做法是再加一层 L1：lua_shared_dict 提供跨 worker 共享内存键值。它的语义更接近固定容量的共享 Map：set 在内存不足时可能强制淘汰有效项并返回 forcible；safe_set 则不强制淘汰，失败即返回。get_stale 可读取过期值，用于 stale-while-revalidate。完整大响应通常仍放 proxy_cache、Redis 或对象存储，shared dict 只放热点摘要、压缩体、重定向元数据或短时防击穿标记。

与前端已有体系对比：浏览器也有 memory cache 与 disk cache、Service Worker Cache Storage 的分层，热点资源驻留内存，冷资源落盘，存储压力触发淘汰。相同点是都用分层存储、TTL/策略失效、空间不足时淘汰。不同点是 Nginx 是服务端共享缓存，服务多个客户端，必须处理共享键空间、Vary、Authorization、no-store/private 等约束；前端缓存通常是单用户代理、按 origin 配额、可被用户清除。OpenResty shared dict 类似跨 worker 的 Map 加 TTL，但没有浏览器那样的完整 HTTP 语义，HTTP 语义仍要由 Cache-Control、ETag、Vary 和业务键设计保证。

### 3. 基础代码与实战验证
```text
http {
  # L2：Nginx 磁盘缓存。keys_zone 只存键与元数据，不存响应体。
  proxy_cache_path /var/cache/nginx/http levels=1:2 keys_zone=http_cache_meta:16m max_size=10g inactive=30m use_temp_path=off;
  lua_shared_dict http_swap_l1 64m;

  server {
    location /api/ai/ {
      proxy_cache http_cache_meta;
      # 缓存键决定隔离粒度；高基数头会放大索引内存并降低命中率。
      proxy_cache_key '$scheme|$host|$request_uri|$http_x_cache_bucket';
      proxy_cache_valid 200 60s;
      # 上游错误或更新期间可返回旧缓存，降低尾延迟。
      proxy_cache_use_stale error timeout updating http_500 http_502;
      proxy_cache_background_update on;
      proxy_cache_lock on;
      proxy_pass http://ai_backend;
    }

    location /api/model {
      content_by_lua_block {
        local dict = ngx.shared.http_swap_l1
        local key = ngx.var.uri .. ':' .. (ngx.var.arg_q or '')
        -- get_stale 允许拿到过期值，作为 stale 候选。
        local val, _, stale = dict:get_stale(key)
        if val and not stale then
          ngx.print(val)
          return
        end

        -- miss 或 stale：访问 L2/上游。真实实现可用 cosocket、proxy_cache 或外部缓存。
        local body = fetch_from_l2_or_upstream(key)

        -- set 内存不足时可能按 LRU 策略强制腾挪有效项；forcible 表示发生淘汰。
        local ok, err, forcible = dict:set(key, body, 30)
        if not ok and err == 'no memory' then
          -- safe_set 不强制淘汰有效项；失败时宁可放弃 L1，也不破坏既有热项。
          dict:safe_set(key, body, 30)
        end

        if val and stale then
          -- stale-while-revalidate：先返回旧值，后台更新，避免同步等待上游。
          ngx.timer.at(0, function()
            update_l2_and_l1(key)
          end)
          ngx.print(val)
        else
          ngx.print(body)
        end
      }
    }
  }
}
```

### 4. 常见误区与进阶思考
误区一：把 proxy_cache_path 的 max_size、inactive 当成内存 LRU。max_size 主要约束磁盘缓存体积，inactive 是未访问即清理，不代表 HTTP 新鲜度；keys_zone 也不是响应体缓存，扩大它只提升元数据容量，不会让大响应驻留内存。
误区二：把高基数身份、Cookie、完整查询串直接塞入缓存键，或在 lua_shared_dict 中存完整大 body。前者造成键爆炸、命中率下降和共享内存耗尽；后者让 L1 承担 L2 职责，导致频繁淘汰、写放大与跨进程锁竞争。
思考题：如果平均响应 2 MiB，峰值 QPS 下内存有限，你应如何让 lua_shared_dict、proxy_cache 与 stale-while-revalidate 分工？请用命中率、forcible 比例、磁盘 I/O、上游 RT 和过期回源风暴说明你的淘汰策略是否正确。
