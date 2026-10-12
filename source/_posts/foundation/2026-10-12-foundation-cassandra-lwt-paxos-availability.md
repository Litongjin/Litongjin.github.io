---
title: "每日基础技术总结 · 2026-10-12 · Cassandra LWT (Lightweight Transactions) PAXOS 协议的可用性权衡"
date: 2026-10-12 10:07:14
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-12 · Cassandra LWT (Lightweight Transactions) PAXOS 协议的可用性权衡

## 📚 今日主题

> **Cassandra LWT (Lightweight Transactions) PAXOS 协议的可用性权衡**（后端基础）

### 1. 核心概念速览
Cassandra LWT（Lightweight Transactions）是 Cassandra 基于 Paxos 共识协议实现的单分区条件写入机制，典型语法为 INSERT IF NOT EXISTS、UPDATE IF condition、DELETE IF condition。其本质是分布式 compare-and-set：在写入前先通过 Paxos 对一个分区键的当前值达成多数派共识，再由协调者基于共识值判断 IF 条件是否成立，成立才提交新值。

它解决的核心问题是：普通 Cassandra 写入只保证最终一致，协调者与副本之间没有“先检查后写入”的原子性，并发条件写入会因到达顺序不同而丢失约束；LWT 通过 Paxos 将同一个分区的并发条件更新串行化，提供单分区线性一致性。

在整个分布式存储体系中，LWT 处于 CAP/PACELC 的 CP 端：发生分区时要求多数派，否则失败；正常无分区时，它以更高延迟和更低吞吐换取单分区共识。专业工程师必须掌握，因为它揭示了高级一致性功能与高可用、低延迟之间的根本权衡，也是理解 Etcd、ZooKeeper、Spanner 等共识系统的基础。

### 2. 底层原理剖析
普通 Cassandra 写入中，协调者只需按一致性级别（如 ONE、QUORUM）将 mutation 发送到副本，副本直接应用并 ack；两个并发 UPDATE IF 可能同时读取旧值并都尝试写入，最后后写覆盖先写，破坏条件约束。

LWT 将一次条件写拆成如下流程：
1. 协调者为该分区生成一个单调递增的 Paxos ballot，并向所有副本发送 prepare(ballot)。
2. 副本若未承诺更高 ballot，则承诺拒绝更低 ballot，并返回已接受的最高 ballot 及其值（若有）。
3. 协调者收到多数派 promise 后，以其中最高已接受值作为当前共识值；若不存在已接受值，则以基础表当前值作为判断依据。然后评估 IF 条件：不满足则返回 applied=false 和当前值；满足则构造新值。
4. 协调者向多数派发送 accept(ballot, new_value)。副本接受后将 ballot、value 持久化到 system.paxos 表并返回 accepted。
5. 协调者收到多数派 accepted 后，向副本发送 commit，将共识后的 mutation 应用到基础表，并向客户端返回 applied=true。

可用性权衡来自 majority quorum：RF=3 时必须收到 2 个 promise 和 2 个 accepted 才能推进。网络分区导致无法联系多数派时，LWT 直接失败，不会像普通写入 CL=ONE 那样继续返回成功。延迟至少包含 prepare/promise 与 propose/accept 两轮多数派 RTT，远超普通写入的一轮 ack；同时 system.paxos 的读写会产生额外写放大。

与前端已有知识体系的对比：前端单线程中的互斥、Atomics.compareExchange 等 CAS 原语依赖共享内存和唯一执行线程，无法直接解决多副本并发；HTTP 中的 ETag/If-Match 是乐观并发控制，冲突检测在客户端重试，而 LWT 将冲突仲裁下沉到服务端 Paxos 多数派，单次请求即可确定是否应用，但代价是网络往返和可用性。

### 3. 基础代码与实战验证
```text
-- 建表与基础数据
CREATE TABLE accounts (
  id UUID PRIMARY KEY,
  balance int
);

-- 普通写入：不检查条件，CL=ONE 也能成功
INSERT INTO accounts (id, balance)
VALUES (123e4567-e89b-12d3-a456-426614174000, 100);

-- LWT 条件写入：仅当 balance 仍为 100 时才改为 90
UPDATE accounts
SET balance = 90
WHERE id = 123e4567-e89b-12d3-a456-426614174000
IF balance = 100;

-- 返回值示例：
-- applied=true 表示 Paxos 共识后条件成立且写入已应用
-- applied=false 表示条件不成立，同时返回当前 balance，可避免客户端盲目重试

-- 验证并发与可用性：
-- 3 节点集群，RF=3。两个客户端同时对同一 id 执行 UPDATE ... IF balance=100。
-- 结果：一条请求 applied=true，另一条请求 applied=false 且返回 balance=90。
-- 原因：Paxos prepare/propose 将两个条件更新串行化，后到者看到已共识的 90，不满足 IF balance=100。

-- 可用性权衡验证：
-- 停掉 2 个节点后，普通写入 CL=ONE 仍可成功（协调者存活即可）。
-- 相同条件下，LWT 因无法收到多数派 promise/accept 而超时或返回 unavailable。
-- 可在 cqlsh 开启 TRACING ON，观察 prepare/promise/propose/accept 的消息阶段。
```

### 4. 常见误区与进阶思考
常见误区：
1. 把 LWT 当成传统 ACID 事务或跨分区事务。LWT 仅对单个 partition key 的条件写提供线性一致性，不具备跨多个分区的原子提交和隔离能力。若把多分区业务逻辑拆成多个 LWT，中间失败仍可能产生部分写入。
2. 认为 LWT 可以像普通写入一样按一致性级别降级。LWT 硬性依赖多数派 Paxos 共识，不能指定 CL=ONE 或以 ANY 成功；分区导致多数派不可达时，LWT 失败，而普通写入仍可返回成功。即使客户端超时重试，也要注意结果可能已 applied，需要幂等设计。

思考题：在 RF=3、热点分区高竞争场景下，多个客户端各自生成随机 ballot 并发发起 LWT，为什么可能出现活锁和吞吐骤降？如果改为由全局单调递增的 ballot 源统一分配 ballot，能否彻底消除高竞争下的活锁？这种设计又会给可用性和延迟引入什么新的瓶颈？
