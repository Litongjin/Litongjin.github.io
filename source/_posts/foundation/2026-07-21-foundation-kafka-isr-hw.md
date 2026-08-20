---
title: "每日基础技术总结 · 2026-07-21 · Kafka 分区副本的 ISR 收缩与 HW 截断机制"
date: 2026-07-21 08:00:00
categories: [技术分享]
tags: ["技术分享", "架构与设计"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-21 · Kafka 分区副本的 ISR 收缩与 HW 截断机制

## 📚 今日主题

> **Kafka 分区副本的 ISR 收缩与 HW 截断机制**（架构与设计）

### 1. 核心概念速览
ISR（In-Sync Replica）是 Kafka 分区副本集合中与 Leader 保持同步的副本子集，其同步性由 `replica.lag.time.max.ms` 控制，本质是 Leader 维护的一份动态成员清单。ISR 收缩是指当 Follower 副本的 fetch 请求滞后超过阈值或长时间未拉取时，Leader 将其从 ISR 中移除；HW（High Watermark）是分区中已提交消息的边界，所有副本只能读取到 HW 之前的数据，HW 的推进受 ISR 中最慢副本的 LEO（Log End Offset）约束。该机制解决的是分布式日志系统在副本故障、网络分区、Leader 切换等场景下的一致性与可用性权衡问题——它不追求所有副本强一致，而是保证 ISR 内副本的最终一致，并允许丢数据以换取可用性（unclean leader election 可配置）。在整个分布式存储体系（如 Raft、Paxos）中，Kafka 的 ISR 是一种“可降级”的多数派替代方案，它用同步副本子集代替法定人数，从而在副本数较多时提高写入效率。专业工程师必须掌握它，因为这是 Kafka 数据可靠性、消息不丢失语义、消费端可见性以及集群故障恢复行为的根本依据，任何生产事故排查和性能调优都绕不开 ISR/HW 的演进规则。

### 2. 底层原理剖析
核心机制围绕两条独立指针：每个副本维护 LEO（下一条待写入消息的偏移量）和 HW（已提交偏移量），Leader 额外维护每个 Follower 的 LEO 副本。

1. ISR 收缩条件：Leader 周期性检查 Follower 的拉取请求时间与拉取进度。若 Follower 在 `replica.lag.time.max.ms` 内未发起 fetch，或其 LEO 落后 Leader LEO 超过阈值（由 broker 配置的 `replica.lag.max.messages`，现已废弃，主要依赖时间），则 Leader 将其移出 ISR。收缩操作由 Leader 线程在更新 ISR 时完成，并写入 `/brokers/topics/[topic]/partitions/[partition]/state` 的 znode（由 controller 接收并广播）。

2. HW 更新与传播：Leader 收到 Follower 的 fetch 请求时，会计算当前可提交的偏移量：`newHW = min(leaderLEO, min(followerLEO in ISR))`。当该值大于当前 HW 时更新。Follower 在 fetch 响应中携带 HW，Follower 根据该值截断自己的日志（实际上 Follower 只截断到 HW，因为高于 HW 的数据可能未提交）。

3. 截断机制场景：Leader 故障后，新的 Leader 从 ISR 中选出。所有 Follower 将自己的 LEO 与新的 Leader 的 HW 对比，若本地 LEO 大于新 Leader 的 HW，则截断到 HW；若小于，则从 HW 开始向新 Leader 同步。但这里存在经典的时间差问题：旧 Leader 可能有一些未提交的消息，这些消息的偏移量高于 HW，但旧 Leader 在故障前已经将它们返回给消费者（如果 `acks=0` 或 `acks=1`），造成数据丢失或重复。

4. 精确过程伪代码：
```
// Leader 端定期检查
for follower in allFollowers:
    if now - follower.lastFetchTime > replica.lag.time.max.ms:
        removeFromISR(follower)
    else if follower.leo < leader.leo - replica.lag.max.messages:
        removeFromISR(follower)

// 处理 fetch 请求
onFetch(follower, fetchOffset):
    follower.leo = fetchOffset
    follower.lastFetchTime = now
    if follower in ISR:
        newHW = min(leader.leo, min(f.leo for f in ISR))
        if newHW > currentHW:
            currentHW = newHW
    return (messages up to currentHW, currentHW)

// 新 Leader 启动时
for follower in allFollowers:
    if follower.leo > leader.hw:
        follower.truncateTo(leader.hw)
    follower.startFetching(from = leader.hw)
```

与前端概念的对比：这与前端状态管理中的“乐观更新”和“回滚”有本质相似。前端在多个客户端同时操作同一份数据时，通常以某个时间戳或版本号作为提交边界（类似 HW），超过边界的未确认变更会在冲突时被丢弃（类似截断）。而 ISR 收缩则类似于前端“在线状态”的判定——若一个客户端心跳超时，服务端将其从活跃用户列表中剔除。但 Kafka 的机制更严格，因为它涉及持久化存储的物理日志截断，且截断后的数据不可恢复（除非从其他副本重新复制）。Java 接口与 TS 接口的区别在于：Java 接口是编译期类型契约，TS 接口是结构化类型系统的运行时形态（实际上 TS 接口在编译后消失）。类比 Kafka，ISR 是运行时动态集合，而非静态配置——它更像是 TS 中通过类型守卫动态收窄的联合类型，而不是声明时的固定元组。

### 3. 基础代码与实战验证
由于 Kafka 协议较为复杂，这里提供一段极简的 Python 伪代码模拟 ISR 收缩与 HW 截断的核心逻辑，不依赖任何框架，直接演示状态机：

```python
import time

class Replica:
    def __init__(self, id, leo):
        self.id = id
        self.leo = leo
        self.last_fetch_time = time.time()

class LeaderPartition:
    def __init__(self, leader_leo):
        self.leader_leo = leader_leo
        self.hw = 0
        self.isr = {}

    def add_follower(self, replica):
        self.isr[replica.id] = replica

    def check_isr(self, max_lag_ms):
        # 模拟周期性检查：移除心跳超时或落后过多的 follower
        now = time.time()
        expired = []
        for fid, rep in self.isr.items():
            if (now - rep.last_fetch_time) * 1000 > max_lag_ms:
                expired.append(fid)
            # 实际 Kafka 还检查落后消息数，此处简化
        for fid in expired:
            del self.isr[fid]
            print(f"Replica {fid} removed from ISR")

    def on_fetch(self, fid, fetch_offset):
        # follower 发起 fetch，更新其 leo 和 last_fetch_time
        rep = self.isr.get(fid)
        if rep:
            rep.leo = fetch_offset
            rep.last_fetch_time = time.time()
            # 重新计算 HW：ISR 中最小的 LEO 与 Leader LEO 的最小值
            min_follower_leo = min([r.leo for r in self.isr.values()], default=self.leader_leo)
            new_hw = min(self.leader_leo, min_follower_leo)
            if new_hw > self.hw:
                self.hw = new_hw
                print(f"HW advanced to {self.hw}")

    def leader_failover(self, new_leader_id):
        # 新 leader 产生后，所有 follower 截断到新 leader 的 HW
        # 假设新 leader 的 HW 等于当前 HW（简化）
        new_hw = self.hw
        for rep in self.isr.values():
            if rep.leo > new_hw:
                rep.leo = new_hw
                print(f"Replica {rep.id} truncated to {new_hw}")
```

关键行注释：
- `min_follower_leo = min([r.leo for r in self.isr.values()], default=self.leader_leo)`：HW 推进必须等待 ISR 中最慢的副本追上，否则 HW 不能超过它，这是“不丢已提交消息”的保证。
- `del self.isr[fid]`：收缩 ISR 后，HW 可能被推进（因为最慢副本被移除），这等价于牺牲该副本的数据来提升可用性。
- `if rep.leo > new_hw: rep.leo = new_hw`：截断只发生在 LEO 高于 HW 时，这确保了日志单调性，但可能导致已发送给消费者的消息丢失（如果该副本是旧 Leader）。

### 4. 常见误区与进阶思考
误区一：认为 HW 是“所有副本中最小的 LEO”。实际上 HW 是 ISR 中最小 LEO 与 Leader LEO 的最小值。ISR 之外的副本 LEO 不参与 HW 计算，因此一个副本被移出 ISR 后，Leader 的 HW 可以立刻前进，但这意味着该副本的数据不再被保证。

误区二：认为 ISR 收缩是永久的，或者被移出的副本无法回来。实际上 Kafka 的 ISR 是动态的：当落后的 Follower 追上 Leader 的 LEO 后，会重新被加回 ISR。但加回 ISR 时不会触发 HW 回退（HW 只前进不后退），因此新加入的副本必须从 HW 之前开始同步，如果其 LEO 大于 HW（理论上不可能，因为被移出后可能继续从 Leader 拉取数据导致 LEO 超过 HW？实际 Follower 会截断到 HW，所以不会超过），但可能存在未截断的情况，此时会先截断再同步。

思考题：假设一个分区有 3 个副本，Leader A 的 LEO=100，HW=80，ISR 中有 B（LEO=80）和 C（LEO=100）。此时 C 网络闪断，A 将其移出 ISR。随后 A 继续写入 20 条消息，LEO=120，B 同步到 100，此时 HW 推进到 100。然后 A 宕机，B 被选为新 Leader。请问：C 恢复后重新加入 ISR 时，它的日志如何处理？是否会丢失 A 在 C 被移出后写入的那 20 条消息？请结合 ISR 收缩与 HW 截断机制解释。提示：C 的 LEO 可能大于 B 的 HW，也可能小于，取决于 C 在闪断期间是否通过其他途径获取了数据（实际上不可能，因为只有 Leader 能写入，C 只能从 A 拉取）。C 在恢复后需要从新 Leader B 开始拉取，但 B 的 HW 是多少？B 的 LEO 又是多少？C 是否需要截断？这个问题的答案揭示了 Kafka 在“消息不丢失”与“精确一次”之间的根本权衡。
