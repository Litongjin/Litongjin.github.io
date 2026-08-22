---
title: "每日基础技术总结 · 2026-08-23 · NoSQL 与 Redis 基础"
date: 2026-08-23 06:55:52
categories: [技术分享]
tags: ["技术分享", "后端基础（Node.js / Java / Python）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-23 · NoSQL 与 Redis 基础

## 📚 今日主题

> **NoSQL 与 Redis 基础**（后端基础（Node.js / Java / Python））

### 1. 核心概念速览
NoSQL 是对传统关系型数据库（RDBMS）的范式突破，核心本质是放弃或弱化『固定Schema、ACID事务、SQL查询』这三大关系模型支柱，以换取水平扩展性、高吞吐或特定数据模型下的操作效率。Redis是NoSQL家族中基于内存的键值存储系统，其本质是一个单线程事件循环驱动的、支持多种数据结构（String、Hash、List、Set、ZSet等）的远程字典服务。它解决的问题是：在低延迟（亚毫秒级）、高并发场景下，提供缓存、计数器、分布式锁、消息队列、排行榜等能力。在计算机体系中的位置：处于应用层与持久化存储之间，作为数据访问的加速层或临时状态层，是分布式系统中支撑『状态』的高性能中间件。专业工程师必须掌握它，因为后端系统中80%的性能瓶颈在I/O，而Redis用内存+高效协议将I/O延迟压到极限，同时它的数据结构和原子操作直接决定了分布式算法的实现方式；不理解其单线程模型和持久化机制，就无法正确评估其容量、可靠性和故障影响。

### 2. 底层原理剖析
Redis的底层核心是『单线程事件循环 + 非阻塞I/O』。所有命令（包括复杂操作如ZADD、LPOP）在同一个线程内串行执行，因此天然原子，不需要锁。其I/O复用基于epoll（Linux）或kqueue（BSD），将多个客户端连接的文件描述符注册到事件循环中，当数据可读/可写时才触发回调。伪代码如下：
```
while (1) {
    events = epoll_wait(server_fd, timeout); // 等待就绪事件
    for (ev in events) {
        if (ev is readable) {
            read_request(ev.fd);            // 解析协议，得到命令
            execute_command(cmd);           // 执行命令，修改内存数据结构
            write_response(ev.fd);          // 写回结果
        }
    }
}
```
内存数据结构是Redis的高效基础：String用SDS（简单动态字符串）避免C字符串的O(n)长度计算和二进制安全问题；Hash用ziplist/quicklist/listpack在元素少时节省内存，元素多时转为hashtable；ZSet用跳跃表+哈希表实现O(logN)的排序和O(1)的分数查询。持久化机制RDB（快照）和AOF（追加日志）本质是内存状态的外化，RDB通过fork子进程写快照，利用COW（写时复制）保证主线程不阻塞；AOF将每个写命令追加到文件，通过fsync策略控制数据丢失窗口。与前端已有概念的对比：Redis的『数据结构』好比JavaScript中的Map、Set、数组，但Redis的数据结构是服务端共享、跨进程、带原子操作命令（如INCR、HSET）的，而前端的是客户端内存私有、无原子跨请求保证的。Redis的『单线程』类似于浏览器JavaScript主线程的事件循环——一个任务执行时不能被其他任务打断，但Node.js的异步I/O依赖libuv线程池，而Redis所有I/O和计算都在主线程，纯CPU操作如KEYS命令会阻塞整个服务，这解释了为何生产环境禁用KEYS。Redis的『过期键』机制类似于前端LocalStorage的过期时间，但Redis是惰性删除+定期删除的混合策略：访问时检查过期则删除，同时周期性随机抽取部分键删除过期键，以控制内存占用。

### 3. 基础代码与实战验证
以下用Node.js原生net模块实现一个极简Redis式服务（仅支持SET/GET），验证单线程事件循环和协议解析本质：
```
const net = require('net');
const store = new Map(); // 内存字典

const server = net.createServer((socket) => {
    socket.setEncoding('utf8');
    let buffer = '';

    socket.on('data', (chunk) => {
        buffer += chunk;
        // Redis协议（RESP）以CRLF分隔，这里简化：每行一条命令
        const lines = buffer.split('\r\n');
        buffer = lines.pop(); // 保留不完整的尾部

        for (const line of lines) {
            // 命令形如: SET key value 或 GET key
            const [cmd, key, value] = line.split(' ');
            if (cmd === 'SET') {
                store.set(key, value);
                socket.write('+OK\r\n'); // RESP简单字符串
            } else if (cmd === 'GET') {
                const val = store.get(key);
                if (val === undefined) {
                    socket.write('$-1\r\n'); // RESP空批量
                } else {
                    socket.write(`$${val.length}\r\n${val}\r\n`); // 批量字符串
                }
            }
        }
    });
});

server.listen(6379, () => console.log('Mini Redis on 6379'));
```
关键行注释：
- `store = new Map()`：底层就是哈希表，O(1)读写，Redis的String就是类似结构。
- `buffer += chunk`：TCP粘包处理，必须缓存不完整数据，Redis的输入缓冲也是同理。
- `socket.on('data')`：事件驱动，同一线程处理所有连接，无需锁。
- `socket.write`：直接写回，Redis的回复也是基于事件循环异步写，但这里简化成同步写。
实际Redis用C实现，但原理一致。此代码验证了单线程处理多个客户端的关键：所有逻辑在data回调中执行，JavaScript的事件循环天然保证串行。

### 4. 常见误区与进阶思考
误区一：认为Redis是数据库，把数据当唯一事实源。Redis本质是带可选持久化的缓存/加速层，其持久化默认不是强一致的：RDB可能丢失最近一次快照后的数据，AOF即使每秒fsync也可能丢失1秒数据。专业工程师若把关键业务数据只放Redis，宕机即丢数据。正确认知：Redis应作为『热数据层』，底层必须配合数据库或消息日志保证持久性。误区二：忽略单线程的『慢命令』风险。因为所有命令串行执行，一条O(N)命令（如KEYS、SMEMBERS、HGETALL大key）会阻塞全部客户端。前端工程师容易类比JS中的while(true)阻塞UI，但Redis的阻塞影响的是整个后端集群的请求。正确做法是使用SCAN系列游标命令，或拆分大key。思考题：在Redis单线程模型下，如何实现一个分布式锁的『续期』机制（即锁的TTL快到期时自动延长），要求不能使用额外的定时器线程，且必须保证当持有锁的客户端崩溃时不产生死锁？请从事件循环和键过期机制的角度，设计一个不依赖外部脚本的原子方案。
