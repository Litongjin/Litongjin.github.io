---
title: "每日基础技术总结 · 2026-05-29 · etcd 的 Raft 协议与领导者选举"
date: 2026-05-29 08:00:00
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-05-29 · etcd 的 Raft 协议与领导者选举

## 📚 今日主题

> **etcd 的 Raft 协议与领导者选举**（后端基础）

### 1. 核心概念速览
etcd 是基于 Raft 共识算法实现的分布式键值存储系统。Raft 是一种用于管理复制状态机的共识协议，它将一致性问题分解为领导者选举、日志复制、安全性三个子问题。其本质是通过多数派投票和日志匹配机制，在异步网络且存在故障的节点间达成确定性状态一致。领导者选举是 Raft 中启动和维护共识的入口：任何节点在选举超时后发起投票，获得多数派（N/2+1）选票者成为领导者，负责接收客户端请求并复制日志。该机制解决的核心问题是：在没有全局时钟和中心化协调者的情况下，如何保证分布式系统各节点状态收敛且唯一。它在整个后端体系中位于分布式系统基础理论层，是理解 etcd、Kubernetes、Consul 等一致性基础设施的基石。专业工程师必须掌握它，因为现代分布式系统的可用性、一致性和容错性最终都依赖这类底层协议，而不仅是 API 使用；不掌握底层机制，就无法正确设计系统、排查脑裂、日志不一致等疑难问题。

### 2. 底层原理剖析
Raft 的领导者选举基于任期（Term）和随机超时（Randomized Timeout）机制。每个节点有 Follower、Candidate、Leader 三种状态，初始均为 Follower。

选举流程（精确伪代码）：
```
每个节点维护：currentTerm, votedFor, log[]
Follower 收到来自 Leader 的心跳（AppendEntries）或合法投票请求，则重置选举超时计时器。
若选举超时（150-300ms 随机），则 Follower 转为 Candidate：
  currentTerm++
  votedFor = self
  发送 RequestVote RPC 给所有其他节点，参数包含 currentTerm、lastLogIndex、lastLogTerm
Candidate 收到响应：
  若收到多数派选票 -> 成为 Leader，立即发送心跳（AppendEntries，空日志）维持权威
  若收到更高任期 -> 转为 Follower，更新 currentTerm
  若选举超时 -> 重新发起新任期选举
其他节点收到 RequestVote：
  仅当请求任期 >= 当前任期，且 votedFor 为空或等于请求者，且候选人的日志至少与自己一样新（term 较大或 term 相同但 index 较大），才投票，并重置自己的选举超时
```

底层机制关键点：
1. 随机超时：避免多个 Candidate 同时发起选举导致选票分裂。每个节点超时值独立随机，保证大概率只有少数节点先发起。
2. 任期递增：Term 是一个单调递增的逻辑时钟，用于区分陈旧消息。所有 RPC 都携带任期，接收方发现任期比自己大则立即转为 Follower，比自己小则拒绝。
3. 日志新旧比较：投票时要求候选人的日志至少与自己一样新，保证日志完整性。这防止一个日志落后的节点当选后覆盖已提交日志，从而保证安全性（Leader Completeness）。
4. 心跳（AppendEntries）双向作用：既是日志复制通道，也是领导者的续任信号。Follower 收到心跳后重置选举超时，从而抑制新选举。
5. 多数派原则：选举和日志提交都必须获得多数派确认。因此即使部分节点故障（最多 N/2 个），系统仍可运行，但无法容忍脑裂——因为两个分区各自无法同时获得多数派。

与前端知识体系的对比：
- Raft 的任期类似前端版本控制中的 commit hash 的单调性，但它是逻辑时钟而非内容哈希；类似 TypeScript 的 `readonly` 约束，但更底层——它约束的是节点状态转换的合法性。
- Raft 的多数派投票类似前端状态管理（如 Redux）中的 reducer 必须纯函数，但它解决的是跨节点的同一性，而非单进程内的确定性。
- Raft 的日志复制类似前端事件总线的广播，但需要确认和持久化，且存在乱序、丢失、重复等网络问题，不像前端事件在同进程内可靠有序。
- Java 接口与 TS 接口的区别是编译期类型约束，而 Raft 的协议是运行期网络约束；前者是静态约定，后者是动态共识，两者在抽象层级上完全不同。

### 3. 基础代码与实战验证
由于 Raft 涉及网络交互和持久化，这里用极简的 Go 伪代码演示选举核心逻辑（不依赖 etcd 框架），聚焦状态转换与投票条件。

```go
package raft

type Node struct {
    currentTerm int
    votedFor    int   // 候选人 ID，-1 表示未投票
    log         []LogEntry
    role        Role  // Follower, Candidate, Leader
    electionTimeout time.Duration
}

type LogEntry struct {
    Term int
    Index int
    Data interface{}
}

type RequestVoteArgs struct {
    Term         int
    CandidateID  int
    LastLogIndex int
    LastLogTerm  int
}

type RequestVoteReply struct {
    Term        int
    VoteGranted bool
}

// 节点收到投票请求时的处理（核心逻辑）
func (n *Node) RequestVote(args RequestVoteArgs) RequestVoteReply {
    reply := RequestVoteReply{Term: n.currentTerm, VoteGranted: false}

    // 1. 如果请求任期小于当前任期，直接拒绝（陈旧消息）
    if args.Term < n.currentTerm {
        return reply
    }

    // 2. 如果请求任期大于当前任期，立即更新为 Follower（当前节点发现自己的任期落后）
    if args.Term > n.currentTerm {
        n.currentTerm = args.Term
        n.role = Follower
        n.votedFor = -1
    }

    // 3. 检查是否已投票：同一任期只能投一票
    if n.votedFor != -1 && n.votedFor != args.CandidateID {
        return reply
    }

    // 4. 日志新旧检查：候选人日志必须至少与本地一样新
    lastLogIndex := len(n.log) - 1
    lastLogTerm := 0
    if lastLogIndex >= 0 {
        lastLogTerm = n.log[lastLogIndex].Term
    }
    if args.LastLogTerm < lastLogTerm ||
        (args.LastLogTerm == lastLogTerm && args.LastLogIndex < lastLogIndex) {
        return reply
    }

    // 5. 满足所有条件，投票并重置选举超时（防止自己立即变成 Candidate）
    n.votedFor = args.CandidateID
    n.electionTimeout = resetElectionTimeout() // 随机 150-300ms
    reply.VoteGranted = true
    reply.Term = n.currentTerm
    return reply
}

// Candidate 发起选举（定时器触发）
func (n *Node) startElection() {
    n.currentTerm++
    n.votedFor = n.selfID
    n.role = Candidate

    // 重置选举超时，防止选举期间再次超时
    n.electionTimeout = resetElectionTimeout()

    args := RequestVoteArgs{
        Term:         n.currentTerm,
        CandidateID:  n.selfID,
        LastLogIndex: len(n.log) - 1,
        LastLogTerm:  n.log[len(n.log)-1].Term,
    }

    votesGranted := 1 // 自己投自己
    for each peer in peers {
        reply := sendRequestVote(peer, args)
        if reply.Term > n.currentTerm {
            // 发现有更高任期，退位为 Follower
            n.currentTerm = reply.Term
            n.role = Follower
            n.votedFor = -1
            return
        }
        if reply.VoteGranted {
            votesGranted++
            if votesGranted > len(peers)/2 { // 多数派
                n.role = Leader
                // Leader 立即发送空 AppendEntries 心跳以确立权威
                n.broadcastHeartbeat()
                return
            }
        }
    }
    // 未获得多数派，等待随机超时后重新选举（循环）
}
```

关键注释：
- `resetElectionTimeout` 返回随机超时，使各节点在不同时刻发起选举，降低碰撞概率。
- 投票条件中的日志新旧比较，是 Raft 安全性核心：确保新 Leader 包含所有已提交日志。
- 收到更高任期立即降级为 Follower，这是整个协议维持单调性的关键。

### 4. 常见误区与进阶思考
误区一：认为领导者选举是『谁先发起谁当选』。实际上，Raft 的选举不仅仅是靠超时和多数票，日志的新旧程度是硬性门槛。一个日志落后的节点即使先发起选举，其他拥有更新日志的节点也会拒绝投票。很多工程师在排查 etcd 问题时，看到 leader 频繁切换就以为是网络抖动，却忽略了可能是某个节点日志落后触发的新选举。

误区二：混淆『多数派』与『全部节点』。Raft 的提交和选举都只需要多数派（quorum），这意味着少数节点宕机不影响服务。但代价是：如果网络分区导致一个分区包含多数派、另一个包含少数派，只有多数派分区能选举出 Leader 并接受写请求；少数派分区永远无法选举出 Leader，这正是防止双主脑裂的关键。但很多工程师以为 etcd 集群节点越多越稳，实际上节点越多，单次写请求需要确认的副本数越多，性能和可用性反而可能下降。

思考题：考虑一个 5 节点 etcd 集群，当前 Term 为 10，节点 A 是 Leader。此时网络发生分区，节点 A 和节点 B 在一个分区，节点 C、D、E 在另一个分区。如果节点 C、D、E 各自都有完整的日志（与 A 相同），但 B 的日志落后了。请分析：
- 分区后哪个分区能选举出新的 Leader？为什么？
- 如果节点 A 所在分区只能联系到 B，那么 A 是否还能继续处理客户端写请求？为什么？
- 在分区恢复后，B 的日志如何被修复？这体现了 Raft 的哪个机制？
（答案要点：C/D/E 分区有 3 个节点构成多数派，可以选举新 Leader；A/B 分区只有 2 个节点，不足多数派，A 不能提交新日志，因此写请求会失败；分区恢复后，B 通过 AppendEntries 心跳收到新 Leader 的日志，进行日志覆盖（以 Leader 日志为准），体现日志复制的强制一致性。）
