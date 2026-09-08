---
title: "每日基础技术总结 · 2026-09-09 · NoSQL 与 Redis 基础"
date: 2026-09-09 07:02:15
categories: [技术分享]
tags: ["技术分享", "后端基础（Node.js / Java / Python）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-09 · NoSQL 与 Redis 基础

## 📚 今日主题

> **NoSQL 与 Redis 基础**（后端基础（Node.js / Java / Python））

### 1. 核心概念速览
NoSQL 是绕开关系型数据库固定模式（Schema、SQL、ACID 强约束）的一类分布式数据存储系统总称，核心诉求是水平扩展性、高吞吐、灵活数据模型与最终一致性。Redis 是一个基于内存、由网络事件循环驱动的键值数据结构服务器，支持 String/Hash/List/Set/ZSet/Stream/Geo/Bitmap 等结构，所有命令按串行方式原子执行。它在系统架构中的位置处于应用进程与磁盘型数据库之间，作为热数据访问层、分布式协调状态层和异步消息通道；在 AI 系统中通常承担特征缓存、模型状态存储、实时统计与向量相似度检索。专业工程师必须掌握它，因为它不是‘缓存库’，而是一个完整的网络状态机与内存计算引擎，理解它的序列化、IO 多路复用、持久化与一致性语义，才能真正做好无锁并发、缓存穿透、分布式锁与实时流处理。

### 2. 底层原理剖析
Redis 的本质是一个由 C 实现的事件驱动状态机：主线程阻塞在 epoll/kqueue 上监听所有客户端 socket 的可读/可写事件，每当有新请求到达，就读取并解析 RESP（REdis Serialization Protocol）命令，从全局哈希表 dict 中找到 key 对应的对象，派发到对应类型的命令处理函数，执行后把响应写回 socket。所有命令在主线程中串行执行，因此没有传统锁和原子性竞争问题；这种单线程模型与前端 JavaScript 的 event loop 有相似逻辑，但 Redis 没有宏任务/微任务分层，所有任务都只是 IO 事件回调，只有命令执行本身可能阻塞主线程，例如慢命令 KEYS、大集合的 DEL/FLUSHALL。底层数据结构上，key 永远是一个 SDS（Simple Dynamic String），value 是一个 robj，robj 的 encoding 决定实际存储形态：String 底层可能为 int、embstr、raw；Hash 在元素少且值小时采用 listpack/ziplist，超过阈值变成 dict；List 用 quicklist；ZSet 用 skiplist + dict，兼顾 O(log N) 的范围查询与 O(1) 的分数/成员查找。持久化机制中，RDB 通过 fork() 子进程利用 COW（Copy-On-Write）生成全量快照，AOF 以追加协议日志的方式记录每个写命令，并通过后台 rewrite 压缩。主从复制使用 psync 和 replication offset 实现断线增量同步。与前端已有概念对比：TS 的 interface 是编译期结构类型，约束在编译后消失，而 NoSQL 文档或 Redis 的数据类型是运行时真实存在的对象结构，Schema 的缺失等于把数据合法性校验推给应用层，类似运行时鸭子类型与静态接口的差异；浏览器 localStorage 是进程内同步阻塞存储，而 Redis 是跨进程、跨网络的原子操作服务，它的原子性来自服务器端单线程串行化，而不是前端异步模型中的 atomicity；Redis Hash 也不同于 JS 对象，它可以在服务端对单个 field 做原子 HINCRBY，而本地对象没有跨网络的一致语义。

### 3. 基础代码与实战验证
```text
import socket

def encode_cmd(*args):
    # RESP 协议：将命令参数包装成数组，格式为 *<参数个数>\r\n，随后每个参数为 $<字节长度>\r\n<字节>\r\n
    out = b'*' + str(len(args)).encode() + b'\r\n'
    for a in args:
        a = a.encode()
        out += b'$' + str(len(a)).encode() + b'\r\n' + a + b'\r\n'
    return out

s = socket.create_connection(('127.0.0.1', 6379))

# SET foo 100：命令序列化后写入 TCP；内核把数据送到 Redis 的事件可读队列，主事件循环取出并执行
s.sendall(encode_cmd('SET', 'foo', '100'))

# 返回 +OK\r\n，简单字符串以 + 开头，以 CRLF 结尾
print(s.recv(1024).decode())

s.sendall(encode_cmd('INCR', 'foo'))

# Redis 对 foo 对应的 String 对象做原子自增；因为命令在主线程串行执行，INCR 之间天然互斥
print(s.recv(1024).decode())

s.sendall(encode_cmd('TYPE', 'foo'))
# 返回 +string\r\n，验证外层类型；实际编码可能继续查询 OBJECT ENCODING 才能看到 int/embstr/raw 的差异
print(s.recv(1024).decode())
```

### 4. 常见误区与进阶思考
误区一：把 Redis 当成关系型数据库来用。只要开了 RDB/AOF 持久化，就认为数据是强可靠、强一致的，这是错误认知。Redis 默认 AOF fsync 策略是 everysec，崩溃最多可能丢失约一秒写数据；主从复制是异步复制，主节点故障时从节点可能缺少最近写日志。Redis 本质仍是内存优先的数据状态服务，容量受内存约束，适合服务热数据、瞬态状态或可重建的数据，不适合作为唯一事实来源。误区二：把 SETNX + DEL 的锁封装当成生产级分布式锁。分布式锁需要处理锁过期与业务执行时间重叠、可重入性、线程上下文与标识绑定、以及锁释放时的原子 compare-and-delete；单节点 Redis 命令串行执行只能保证单机无竞争，无法解决节点故障、GC 停顿或时钟跳跃带来的安全性问题，必须借助 Lua 脚本、租约续期或 Redlock 等方案。深入思考题：Redis 的事件循环是单线程的，为什么 BLPOP 这类阻塞命令会让某个客户端挂起，却不会阻塞整个 Redis 进程？请从事件状态机、被阻塞 key 的等待队列，以及数据到达后如何唤醒等待者这几个层面解释；进一步思考，如果让你在 Redis 单线程模型上实现一个延迟任务队列，你会如何设计数据结构与事件唤醒机制，避免长时间占用主线程？
