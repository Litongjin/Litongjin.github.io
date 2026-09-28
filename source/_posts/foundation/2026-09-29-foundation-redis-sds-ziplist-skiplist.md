---
title: "每日基础技术总结 · 2026-09-29 · Redis 底层结构：SDS/ziplist/skiplist 演进"
date: 2026-09-29 07:20:08
categories: [技术分享]
tags: ["技术分享", "数据库与缓存进阶"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-29 · Redis 底层结构：SDS/ziplist/skiplist 演进

## 📚 今日主题

> **Redis 底层结构：SDS/ziplist/skiplist 演进**（数据库与缓存进阶）

### 1. 核心概念速览
Redis 的底层数据结构是其在 O(1) 时间复杂度内完成高并发读写、同时保持内存紧凑与有序性的根基。SDS（Simple Dynamic String）是 Redis 对 C 字符串的封装，用于解决二进制不安全、获取长度 O(n)、缓冲区溢出等问题；ziplist 是连续内存块上的紧凑双向链表，用于小规模数据的内存极简存储；skiplist 是一种概率平衡的有序数据结构，作为 zset 的有序索引实现，替代红黑树以简化范围查询与并发控制。三者分别对应字符串、列表/Hash/ZSet 小数据量优化、有序集合大数据量场景，共同构成了 Redis 内存高效与操作稳定的底层基石。专业工程师必须掌握，因为任何命令的时空复杂度、内存碎片成因、持久化文件结构都与这些底层实现直接耦合，也是理解 Redis 演进（如 listpack 替代 ziplist）的前提。

### 2. 底层原理剖析
SDS 本质是带头部元数据的可变字节数组。结构为：len（已用长度）、alloc（分配容量）、flags（类型标志）、buf[]（字节数组，末尾含 '\0' 以兼容 C 字符串）。关键机制：1）O(1) 获取长度，直接读 len；2）二进制安全，以 len 而非空字符判断结束；3）空间预分配与惰性释放，避免频繁 realloc。对比前端：如同 TypeScript 的 string 与 ArrayBuffer 的差异——SDS 显式管理长度和容量，比 JS 引擎的字符串更底层，但思想类似 Uint8Array 的长度属性。\n\nziplist 本质是连续内存块上的序列化结构，由 header（zlbytes/zltail/zllen）、entry 列表、zlend 组成。每个 entry 保存前一个 entry 的长度（prevlen）、编码信息（encoding）和数据。通过维护 prevlen 支持从尾向前遍历，通过 record 偏移量实现双向遍历。紧凑性来自不存储指针对，而是用变长编码和偏移量。机制是：新增/删除需要 memmove 移动后续内存，因此只适合小规模数据（如 list 长度 < 512、hash 字段 < 64）。对比前端：类似 ArrayBuffer 上手动实现链表，以内存连续换取缓存命中率，代价是插入删除 O(n)。\n\nskiplist 本质是多层有序链表。每层是一个有序双向链表，节点层数通过随机函数生成（通常 1/2 概率提升一层）。查询时从最高层开始逐级下降，平均 O(log n)，最坏 O(n)。与红黑树相比，skiplist 优势：实现简单、支持高效范围查询（只需从 prev/next 指针遍历）、并发无锁化改造容易。Redis 的 zset 同时使用 dict（哈希表）存 member->score 和 skiplist 存 score->member，兼顾单点查询与排序。对比前端：skiplist 类似 DOM 的时间片段或虚拟滚动中的分块索引，用多层稀疏索引加速跳转，但这里在内存中进行指针跳跃。

### 3. 基础代码与实战验证
```text
以下为伪代码/结构示意，演示 SDS 长度获取与 ziplist 遍历的核心逻辑。\n\n// SDS 头部结构\nstruct sdshdr {\n    int len;   // 已用长度（不含 '\0'）\n    int alloc; // 分配容量（含额外空间）\n    char flags; // 类型位，决定头部大小\n    char buf[]; // 实际字节数组\n};\n\n// 获取 SDS 字符串长度：直接返回 len，O(1)\nint sdslen(const sds s) {\n    return ((struct sdshdr*)s - 1)->len; // 指针回退到头部取 len\n}\n\n// ziplist entry 结构示意\n// | prevlen (变长) | encoding (变长) | data |\n// 遍历时从 zlbytes+zlend 定位，或从 zltail 反向\n\n// 模拟 ziplist 正向遍历\nunsigned char *zl = ...; // 指向 zlbytes 起始\nunsigned char *p = zl + ZIPLIST_HEADER_SIZE; // 跳过 header\nwhile (p < zl + zlbytes - 1) { // 到 zlend 为止\n    prevlen = decode(p); \n    encoding = decode(p + prevlen_size);\n    p += prevlen_size + encoding_size + data_size; // 跳到下一个 entry\n}\n\n// skiplist 查询伪代码：从最高层开始\nnode *x = zsl->header;\nfor (int i = zsl->level-1; i >= 0; i--) {\n    while (x->level[i].forward && x->level[i].forward->score < score)\n        x = x->level[i].forward; // 向前跳\n    // 保留当前 x 作为第 i 层的候选\n}\n// 最终 x->level[0].forward 即为第一个 score >= 目标值的节点\n\n// 验证：手动构造一个 SDS，观察 len 变化\n// 实际 Redis 中命令：SET key value 底层即创建 SDS，STRLEN key 直接读 len，O(1)。
```

### 4. 常见误区与进阶思考
误区 1：认为 ziplist 是普通链表。实际它是连续内存块，不是链表节点指针连接，因此插入删除需要内存移动，复杂度 O(n)，只适合小规模。若数据量超过阈值触发转换（listpack 已替代），然后才换成双向链表或 hashtable。\n误区 2：认为 skiplist 是平衡树。skiplist 的层数由随机决定，不保证绝对平衡，但概率上接近平衡，最坏情况是 O(n)。Redis 选择它并非因为性能优于红黑树，而是实现简单且范围查询友好。\n\n思考题：给定一个内存受限的 Redis 实例，若大量使用 zset 且 member 数量分布在 100 到 1000 之间，底层 skiplist 的层数期望值是多少？为什么 ZADD 的时间复杂度是 O(log n) 而不是 O(1)？这与 dict 共存的结构如何保证 ZSCORE 依然是 O(1)？
