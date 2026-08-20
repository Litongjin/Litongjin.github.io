---
title: "每日基础技术总结 · 2026-08-07 · ZooKeeper 的 ZAB 协议：崩溃恢复与消息广播阶段"
date: 2026-08-07 08:00:00
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-08-07 · ZooKeeper 的 ZAB 协议：崩溃恢复与消息广播阶段

## 📚 今日主题

> **ZooKeeper 的 ZAB 协议：崩溃恢复与消息广播阶段**（后端基础）

### 1. 核心概念速览
ZAB（ZooKeeper Atomic Broadcast）是 ZooKeeper 实现分布式数据一致性的核心协议，本质是一个支持崩溃恢复的原子广播协议。它解决的问题是：在多副本组成的集群中，如何保证所有副本上的状态变更（事务）以相同的顺序被应用，且在 Leader 崩溃后集群仍能继续对外提供一致性的读写服务。ZAB 将运行过程划分为两个阶段：崩溃恢复（Recovery）和消息广播（Broadcast）。崩溃恢复阶段负责选举出一个新的 Leader，并使其获得集群中已提交事务的最新状态，同时同步未提交但已被提议的事务；消息广播阶段则由 Leader 以两阶段提交的变体方式向所有 Follower 传播事务，确保事务的全局有序性。ZAB 在分布式系统栈中位于一致性协议层，与 Raft 并列，但 ZAB 更强调'恢复后继续广播'的整体流程，而非单纯的状态机复制。专业工程师必须掌握它，因为 ZooKeeper 是众多分布式系统（Kafka、HBase、Dubbo 等）的协调基石，理解 ZAB 才能正确理解这些系统在故障时的行为边界，并能独立分析一致性异常问题。

### 2. 底层原理剖析
ZAB 的核心机制可分解为两个阶段及其状态转换。

1. 消息广播阶段（Broadcast）：
   - Leader 接收到客户端写请求后，将其封装为一个 Proposal（提议），分配一个全局单调递增的 ZXID（ZooKeeper Transaction ID）。ZXID 高 32 位为 epoch（纪元，Leader 任期编号），低 32 位为事务序号。
   - Leader 将 Proposal 发送给所有 Follower，Follower 收到后写入本地事务日志并返回 ACK。
   - 当 Leader 收到超过半数 Follower 的 ACK（包括自己）后，发送 COMMIT 消息给所有 Follower，Follower 提交事务。
   - 该过程本质上是两阶段提交的简化版：只要求多数派确认即可提交，无需等待所有节点。

2. 崩溃恢复阶段（Recovery）：
   - 当 Leader 崩溃或多数派节点无法感知 Leader 时，进入崩溃恢复。该阶段的核心目标是选出一个拥有最新已提交事务的节点作为新 Leader，并确保集群状态一致。
   - 选举过程基于 ZXID：每个参与选举的节点在投票中带上自己的 (epoch, zxid)，节点会优先选择 epoch 最大者，其次选择 zxid 最大者（即事务序号最大，代表状态最新）。当某个节点获得超过半数的投票，它就成为新 Leader。
   - 新 Leader 确定后，会与所有 Follower 建立连接，并发送 NEWLEADER 消息，其中包含自己的最新历史事务列表。Follower 收到后，会对比自己与 Leader 的日志，将缺失的已提交事务补齐（同步），然后向 Leader 确认。
   - 当 Leader 收到所有 Follower（或满足半数）的确认后，才广播 COMMIT 使这些事务生效，然后正式进入广播阶段。
   - 关键机制是：在恢复阶段，ZAB 会丢弃那些尚未被多数派确认的 Proposal，因为这类事务可能已经在 Leader 上生成但未提交，而新 Leader 可能没有它们，为了保证一致性，这些未提交的提议必须被舍弃。

3. 与前端概念的对比：
   - 前端中常提到‘事件循环’（Event Loop）与 ZAB 的广播阶段有相似之处：事件循环保证所有任务按顺序执行，ZAB 保证所有事务按 ZXID 顺序应用。但事件循环是单线程串行，ZAB 是跨节点多数派共识。
   - 前端中的‘状态管理’（如 Redux）通过单一 store 和纯 reducer 保证状态可预测，ZAB 通过 Leader 和多数派确认保证多副本状态一致。两者都强调‘顺序’和‘单一事实来源’，但 ZAB 面临的是网络分区、节点故障等不可靠环境，而前端状态管理假设单机内存可靠。
   - ZAB 与 Raft 的差异：Raft 的选举限制为只有日志最全的节点能成为 Leader，而 ZAB 允许任何节点在选举中胜出，但新 Leader 会通过同步其他节点的日志来补齐自身缺失的已提交事务。ZAB 的恢复阶段包含一个‘日志同步’步骤，而 Raft 的 Leader 只负责处理新日志，不主动补齐旧日志（Follower 会追赶）。

### 3. 基础代码与实战验证
由于 ZAB 是分布式协议，不依赖特定框架，用伪代码描述核心逻辑最准确。以下为关键步骤的精确化伪代码：

```
// 节点角色：Leader, Follower, Observer（略）
// ZXID 结构：高32位 epoch，低32位 counter

// --- 广播阶段（Leader 侧） ---
function onClientWrite(request):
    zxid = (currentEpoch, ++counter)   // 生成全局有序 ID
    proposal = new Proposal(zxid, request.data)
    sendToAllFollowers(LEADER_PROPOSAL, proposal)
    addToPendingProposals(zxid)        // 记录未确认提议

function onFollowerAck(followerId, zxid):
    pendingProposal[zxid].acks.add(followerId)
    if pendingProposal[zxid].acks.size() > halfOfCluster:
        sendToAllFollowers(COMMIT, zxid)  // 广播提交
        applyToStateMachine(zxid)          // 提交到本地
        removeFromPending(zxid)

// --- 广播阶段（Follower 侧） ---
function onLeaderProposal(proposal):
    writeToTransactionLog(proposal)  // 先落盘，确保可恢复
    sendAckToLeader(proposal.zxid)   // 返回 ACK

function onLeaderCommit(zxid):
    applyToStateMachine(zxid)        // 应用到内存数据库

// --- 崩溃恢复阶段 ---
function onLeaderElection():
    // 每个节点发送自身 (epoch, zxid) 的投票
    myVote = (myEpoch, myZxid)
    broadcast(myVote)
    while not haveMajorityVote():
        vote = receiveVote()
        // 选 epoch 大者，若 epoch 相同则选 zxid 大者
        if vote.epoch > myVote.epoch || (vote.epoch == myVote.epoch && vote.zxid > myVote.zxid):
            myVote = vote
            broadcast(myVote)
    // 胜出者成为 Leader，其余为 Follower

function onNewLeader(leaderInfo, history):
    // Follower 收到新 Leader 的日志列表
    missingTxns = diff(history, myTransactionLog)
    for each txn in missingTxns:
        sendRequestToLeader(txn.zxid)   // 拉取缺失事务
        writeToTransactionLog(txn)
    sendAckToLeader()                    // 确认同步完成

function onLeaderAfterElection():
    // 新 Leader 等待所有 Follower 确认同步后，发送 COMMIT
    waitForAllAcks()
    for each txn in myHistory:
        sendCommitToAll(txn.zxid)
    // 然后才进入广播阶段，开始处理新的客户端请求
```

关键点：
- ZXID 的单调递增保证了广播阶段的全局顺序。
- Follower 必须先写事务日志再 ACK，确保崩溃后能恢复。
- 恢复阶段新 Leader 必须收到所有 Follower 的同步确认后才提交这些历史事务，避免出现部分节点提交而部分未提交。

### 4. 常见误区与进阶思考
误区 1：认为 ZAB 与 Raft 完全一样。
ZAB 和 Raft 都是领导者驱动的共识协议，但在恢复机制上存在关键差异：Raft 中 Leader 选举后直接开始接受新请求，Follower 通过日志复制机制落后时自然追赶；而 ZAB 在恢复阶段有一个显式的‘同步历史事务并提交’的步骤，新 Leader 必须确保所有 Follower 已经同步了它之前任期的所有已提交事务，然后才进入广播阶段。如果忽略这一区别，在分析 ZooKeeper 节点故障后的行为时会错误地预测数据可见性时序。

误区 2：认为只要多数派 ACK 就立即提交，因此客户端在 COMMIT 前读不到数据。
实际上，Leader 在收到多数派 ACK 后就会向所有 Follower 发送 COMMIT 并自行提交，但 Follower 可能在收到 COMMIT 前就对外提供读服务（ZooKeeper 的 Follower 可处理读请求）。这导致客户端可能在某个 Follower 上读到旧数据，直到该 Follower 完成提交。这是 ZooKeeper 的‘顺序一致性’而非‘线性一致性’的表现。工程师如果误以为 ZooKeeper 所有读都是强一致，就会在设计与测试中踩坑。

思考题：
假设集群中有 5 个节点，当前 Leader 处理事务 T1（ZXID = (1, 1)）并已发送 COMMIT 给 3 个 Follower（共 5 个节点，Leader + 4 Follower），但只有 2 个 Follower 实际提交了 T1，Leader 在此时崩溃。随后选举产生新 Leader，其最新 ZXID 为 (1, 0)（即没有 T1）。根据 ZAB 的恢复规则，T1 应该被丢弃还是被恢复？请结合多数派 ACK 的条件，分析在什么情况下 T1 可能被丢弃，并说明 ZooKeeper 如何保证最终一致性。
