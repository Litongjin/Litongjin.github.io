---
title: "每日基础技术总结 · 2026-08-29 · NoSQL 与 Redis 基础"
date: 2026-08-29 06:55:34
categories: [技术分享]
tags: ["技术分享", "后端基础（Node.js / Java / Python）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-29 · NoSQL 与 Redis 基础

## 📚 今日主题

> **NoSQL 与 Redis 基础**（后端基础（Node.js / Java / Python））

### 1. 核心概念速览
定义：NoSQL（Not Only SQL）是一类摒弃固定关系模型与强 ACID 约束的分布式数据存储系统的统称，核心特征是 Schema-less 数据模型、水平可扩展、可调一致性；常见分类为键值型（Redis）、文档型（MongoDB）、列族型（HBase/Cassandra）、图型（Neo4j）。Redis 本质是一个驻留内存的数据结构服务器：它通过单线程事件循环串行处理命令，以 key-value 为组织单位，内置 String/Hash/List/Set/ZSet/Stream 等数据结构，并提供 RDB（fork+COW 内存快照）与 AOF（追加写日志）两种持久化路径。它解决的问题是存储层级间的时延矛盾——磁盘型数据库单次 IO 在毫秒级，而 Redis 内存访问在亚毫秒级，因此被用作热数据缓存、计数器、分布式锁、限流、消息队列等低延迟组件。在计算机体系中，它位于应用进程与持久化存储之间，属于数据访问路径中的热数据层；在 AI 体系中，它承担特征缓存、推理结果缓存、在线流量的计数与限流。专业工程师必须掌握它，因为任何高并发系统的核心都是把数据放到正确的存储层级中，而 Redis 正是这个层级中最普遍的载体；不理解其底层机制，就无法正确评估可靠性、一致性与性能边界。

### 2. 底层原理剖析
底层原理按五层拆解。一、内存数据结构层：Redis 的 value 是具备类型的数据结构而非任意字节。String 底层为 SDS（带长度字段的动态字符串），解决 C 字符串取长度 O(n) 与缓冲区溢出问题；Hash 底层为 dict（双哈希表渐进式 rehash）；ZSet 底层为 skiplist + dict 组合，dict 负责 O(1) 分值定位，skiplist 负责 O(log N) 范围有序遍历；当元素少或元素短时自动降级为 listpack 紧凑编码。这是 Redis 被称为数据结构服务器的根本原因。
二、单线程事件循环与原子性：主线程基于 aeEventLoop 封装 epoll 多路复用，执行流程为：
aeMain(loop):
  while loop 未停止:
    events = aeApiPoll(loop, timeout)
    遍历 events: 调用 readQueryFromClient 读取并解析 RESP 请求
    对每个请求调用 processCommand，整条命令执行期间不发生任何线程切换或抢占
    processTimeEvents() 处理过期键主动删除与定时任务
    handleClientsWithPendingWrites() 将响应写回客户端
命令串行执行意味着每条命令天然原子，这是用单线程消解竞争的无锁模型。该模型与前端 JS 事件循环的相同点是都靠单线程处理任务；不同点是 JS 的异步非阻塞模型面向高 IO 并发，而 Redis 的命令执行是同步且必须立即完成的，任何慢命令等价于前端主线程上的 long task。
三、过期与淘汰机制：TTL 存放在独立的 expires 字典中，值为绝对毫秒时间戳；命令入口先 expireIfNeeded 做惰性删除，后台时间事件再抽样主动删除；内存达到 maxmemory 后按 LRU/LFU/volatile-ttl 策略执行逐出。
四、持久化机制：RDB 通过 fork 出子进程，子进程基于操作系统写时复制（COW）遍历共享内存页生成快照文件，主进程继续写服务；AOF 以 RESP 格式追加每一条写命令，appendfsync 配置决定每次写、每秒、还是交给内核刷盘；AOF 重写同样依赖 fork 生成最小命令集。两者本质区别：RDB 丢最后一次快照后的数据，恢复快；AOF 可通过 fsync 频率控制丢失窗口，恢复慢。
五、网络协议 RESP：请求本质是命令与参数构成的字符串数组，前缀即类型：+ 简单字符串、- 错误、: 整数、$ 批量字符串、* 数组。与前端概念对齐：TS 的 interface 是编译期结构契约，约束对象形状的静态类型；RESP 是运行期字节级契约，约束进程间消息的解析效率与边界。两者本质都是约定交互双方的格式，但一个服务于类型系统，一个服务于 I/O 管道。与关系型数据库对比：SQL 依赖解析器生成执行计划按索引定位数据，Redis 不做查询计划，所有访问都是显式命令按键寻址；SQL 用锁和 MVCC 保证并发隔离，Redis 用单线程串行化直接消除并发窗口；CAP 语境下单机 Redis 是 CP，Cluster 异步复制下是 AP（主从切换可能丢写命令）。

### 3. 基础代码与实战验证
```text
// 极简验证：不引入任何客户端库，用 Node.js 内置 net 模块手工构造 RESP 协议包
// 前提：本机已启动 redis-server，监听 127.0.0.1:6379
// 验证两个底层事实：
// 1) RESP 是 CRLF 分隔的前缀类型编码协议
// 2) 单线程事件循环使 INCR 天然原子，并发下无竞态

const net = require('net');

// RESP 编码器：把命令参数序列化为请求字节流
// 数组类型前缀为 *，批量字符串前缀为 $，整数响应前缀为 :
function encodeCommand(...args) {
  let msg = '*' + args.length + '\r\n';
  for (const arg of args) {
    const buf = Buffer.from(String(arg));
    msg += '$' + buf.length + '\r\n' + buf.toString('binary') + '\r\n';
  }
  return Buffer.from(msg, 'binary');
}

// ---------- 验证一：并发 INCR 的原子性 ----------
// 100 个独立 TCP 连接同时对同一 key 发 INCR
// Redis 主线程串行执行每条命令，命令之间不可抢占，
// 因此返回值必然严格递增且最终等于 100，这就是无锁原子性
function incrOnce() {
  return new Promise((resolve, reject) => {
    const sock = net.createConnection({ host: '127.0.0.1', port: 6379 });
    sock.on('connect', () => sock.write(encodeCommand('INCR', 'atomic-counter')));
    sock.on('data', (chunk) => {
      // 响应形如 :57\r\n，冒号前缀表示整数（RESP 类型标记）
      const val = parseInt(chunk.toString().slice(1), 10);
      sock.end();
      resolve(val);
    });
    sock.on('error', reject);
  });
}

Promise.all(Array.from({ length: 100 }, incrOnce)).then((vals) => {
  console.log('去重后的返回值数量:', new Set(vals).size); // 必然为 100
  console.log('最终计数值:', Math.max(...vals));          // 必然为 100
});

// ---------- 验证二：TTL 过期与惰性删除 ----------
// SET 携带 PX 参数设置毫秒级过期时间；过期信息存放在独立的 expires 字典中
// 每次命令执行入口调用 expireIfNeeded：命中过期键先删除再返回 nil
const sock2 = net.createConnection({ host: '127.0.0.1', port: 6379 });
let getCount = 0;
sock2.on('connect', () => {
  sock2.write(encodeCommand('SET', 'temp', 'v', 'PX', '1000'));
  setTimeout(() => sock2.write(encodeCommand('GET', 'temp')), 400);
  setTimeout(() => sock2.write(encodeCommand('GET', 'temp')), 1200);
});
sock2.on('data', (chunk) => {
  const text = chunk.toString();
  if (text[0] === '+') {
    console.log('SET:', text.slice(1).trim()); // +OK
  } else if (text[0] === '$') {
    const start = text.indexOf('\r\n') + 2;
    const end = text.lastIndexOf('\r\n');
    console.log('GET:', text.slice(start, end) || 'nil（键已过期并被删除）');
    if (++getCount === 2) sock2.end();
  }
});
```

### 4. 常见误区与进阶思考
误区一：把 Redis 持久化当成可选项，或者反过来把异步复制当作强一致。很多工程师将 Redis 视为缓存而把 appendfsync 设为 no 且不开启 RDB，重启即丢失全部热数据；另一极端是要求主从切换零丢失，却在默认异步复制下产生数据黑洞。本质：Redis 的持久性/性能/可用性由配置权衡——AOF always 每写必 fsync、everysec 最多丢 1 秒、RDB 丢最近快照后的数据；Cluster 异步复制导致故障转移时 lost update。必须用数据可丢性 SLA 反推配置，而不是凭感觉。
误区二：认为所有 Redis 命令都是 O(1) 且单线程足够快。KEYS、SMEMBERS、HGETALL、ZRANGE 全量或范围操作是 O(N) 或 O(log N + M)，在单线程事件循环上执行会像前端主线程的 long task 一样阻塞所有后续请求；一次百万 key 的 KEYS 可把延迟从亚毫秒打到秒级。认知本质：单线程意味着每条命令都必须快速返回，复杂度就是延迟上限，对应前端心智模型是主线程上不能做大规模同步遍历。
思考题：Redis 的 WATCH 命令基于什么机制感知其他客户端对受监视 key 的修改？为什么 MULTI/EXEC 事务在执行期间不会被其他命令插入，而 WATCH 到 EXEC 之间其他客户端的命令却能执行并导致事务被放弃？若 WATCH 一个不存在的 key，在 EXEC 之前该 key 被其他客户端创建，事务会被 abort 吗？答案要点：WATCH 只在客户端记录被监视 key 的版本号，任何修改 key 的命令执行时通过 signalModifiedKey 递增版本；EXEC 时比对版本，不一致则丢弃事务队列并返回 nil。WATCH 不存在的 key 在创建时同样触发版本递增，因此会被 abort。整个检测—执行过程不依赖任何锁，因为 EXEC 命令在单线程上一次性执行完，这体现了乐观锁与悲观锁的本质区别：悲观锁阻塞并发，乐观锁通过版本校验拒绝过期操作。
