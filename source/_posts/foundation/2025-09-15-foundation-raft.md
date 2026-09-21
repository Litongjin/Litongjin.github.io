---
title: "每日基础技术总结 · 2025-09-15 · 分布式一致性：Raft 选主与日志复制"
date: 2025-09-15 20:00:00
categories: [技术分享]
tags: ["技术分享", "分布式与架构设计"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2025-09-15 · 分布式一致性：Raft 选主与日志复制

## 📚 今日主题

> **分布式一致性：Raft 选主与日志复制**（分布式与架构设计）

### 1. 核心概念速览
Raft 是一种通过日志复制实现分布式系统一致性的共识算法，核心解决在多节点网络分区、延迟或故障场景下，如何选举单一领导者（Leader）并确保所有副本状态机最终一致的问题。其本质是将线性化的状态机服务问题分解为两个子问题：Leader 选举（Log Replication 的前提）和日志复制（数据持久化与同步）。在 AI/后端体系中，它是构建高可用存储（如 etcd, CockroachDB）、协调服务（Zookeeper 替代方案）及分布式数据库的基础，专业工程师需掌握它以理解 CAP 理论中的 CP 侧落地机制及如何容忍部分故障。

底层原理剖析：
1. 状态机模型：每个节点处于 Follower, Candidate, Leader 三种状态之一。Follower 被动响应心跳；Candidate 发起选举投票；Leader 处理客户端请求并广播日志条目。
2. Term（任期）机制：单调递增的逻辑时钟，用于解决脑裂冲突。每次选举增加 Term，节点拒绝来自旧 Term 的指令。
3. Leader 选举协议：
   - 启动选举定时器，转为 Candidate，Term +1。
   - 向其他节点发送 RequestVote RPC。
   - 若获得多数派（N/2+1）选票，升级为 Leader。
   - 若收到已知 Leader 的心跳或更高 Term 的 RPC，则回退为 Follower。
4. 日志复制协议：
   - Leader 接收客户端命令，追加到本地日志。
   - 并行发送 AppendEntries RPC (含 prevLogIndex, prevLogTerm, entries) 给 Follower。
   - Follower 验证日志一致性：检查 prevLogIndex 对应的 term 是否匹配；若不匹配，拒绝并回溯指针。
   - Leader 等待多数派成功写入（committed index），通知客户端成功。
5. 强一致性保证：Leader 只有在多数派确认后才提交前置日志，确保被提交的日志永远不会回滚。

对比前端概念：类比 TypeScript 的编译时类型检查 vs JavaScript 的运行时无检查。Raft 的 Term 机制类似于严格的版本控制（Semantic Versioning），强制节点根据版本号（Term）决定接受或拒绝操作，避免‘脏数据’混入。而前端常见的乐观锁（Optimistic Locking）在无冲突时性能好但有冲突重试成本，Raft 是悲观式的强一致，牺牲部分性能换取确定性。

基础代码与实战验证：
// 简化的 Raft 节点状态与选举逻辑伪代码
struct Node {
    CurrentTerm: int
    VotedFor: optional CandidateID
    Log: []Entry
    CommitIndex: int
    LastApplied: int
    State: enum {Follower, Candidate, Leader}
}

func StartElection(node *Node) {
    node.State = Candidate
    node.CurrentTerm++
    node.VotedFor = node.ID
    voteCount := 1 // 自己投一票

    // 广播投票请求
    rpcs := sendRequestVote(node.Nodes, node.CurrentTerm, node.ID)

    for r in rpcs {
        if r.Term > node.CurrentTerm {
            node.CurrentTerm = r.Term
            node.State = Follower
            return // 发现更强领导者，立即回退
        }
        if r.VoteGranted && r.Term == node.CurrentTerm {
            voteCount++
            if voteCount > len(node.Nodes)/2 {
                node.State = Leader
                becomeLeader(node) // 开始心跳与日志复制
            }
        }
    }
}

func AppendEntries(node *Node, req AppendEntriesReq) bool {
    if req.Term < node.CurrentTerm {
        return false // 拒绝过时请求
    }

    // 核心一致性校验：前一项的 Term 必须匹配
    // 这保证了日志序列不会在中间产生分歧
    if node.Log.size() == 0 || node.Log[req.PrevLogIndex].Term != req.PrevLogTerm {
        return false
    }

    // 删除冲突日志（如果存在覆盖）
    if req.Entries.count > 0 {
        deleteConflictLogs(node, req.PrevLogIndex + 1)
        appendEntries(node, req.Entries)
    }

    // 更新 Leader 提交的索引（仅在大多数节点确认后）
    if req.LeaderCommit > node.CommitIndex {
        node.CommitIndex = min(req.LeaderCommit, lastAppliedIndex(node))
    }
    return true
}

常见误区与进阶思考：
1. 误区：认为 Raft 能防止网络分区导致的数据丢失。
   正解：Raft 保证的是安全性（Safety），即已提交的数据不会丢失。但在网络分区期间，如果原 Leader 无法联系多数派，它将停止服务，导致可用性（Availability）丧失。这是 CAP 定理的体现，而非算法缺陷。
2. 误区：混淆 ‘日志追加’ 与 ‘日志提交’。
   正解：AppendEntries 只是尝试复制日志，只有当日志被复制到多数派（Replicated）且其之前的所有日志都被提交后，该条目才算 Committed。前端开发者常忽略这种最终一致性与即时确定性的区别，直接读取未提交数据会导致竞态条件。

深度思考题：
在一个包含 N 个节点的 Raft 集群中，如果网络发生分区，其中一侧有 1 个 Leader 所在节点和 N-1 个 Follower，另一侧有 K 个 Follower（K < N/2）。请问哪一侧能继续提供服务？为什么即使拥有最新的日志，K 个 Follower 也不能选举新的 Leader？请从 ‘多数派 Quorum’ 的定义角度解释。”}
