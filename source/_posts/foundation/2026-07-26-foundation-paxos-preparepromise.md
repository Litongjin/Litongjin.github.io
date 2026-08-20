---
title: "每日基础技术总结 · 2026-07-26 · Paxos 的 Prepare/Promise 阶段与多数派重叠证明"
date: 2026-07-26 08:00:00
categories: [技术分享]
tags: ["技术分享", "架构与设计"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-26 · Paxos 的 Prepare/Promise 阶段与多数派重叠证明

## 📚 今日主题

> **Paxos 的 Prepare/Promise 阶段与多数派重叠证明**（架构与设计）

### 1. 核心概念速览
Paxos 的 Prepare/Promise 阶段是 Basic Paxos 中 Proposer 在发起 Accept 之前，通过一轮“预备”通信确定全局递增提案编号，并收集 Acceptor 已接受过的最高编号提案信息的机制。其本质是：用两轮 RTT（Prepare/Promise 与 Accept/Accepted）将共识问题归约为“单提案编号主导权”问题。Prepare 阶段不是投票，而是“锁定”：Acceptor 承诺不再接受编号小于当前 Prepare 的提案，并返回自身已接受的最大编号提案（若有）。该阶段与多数派重叠证明共同构成 Paxos 安全性的基石：任意两个多数派必有交集，因此任何被选定的值都必然被后续的 Prepare 阶段从至少一个 Acceptor 处获知，从而保证一旦值被选定，后续所有选定值都等于它。在整个分布式系统理论中，Paxos 是状态机复制的核心，也是 Raft 等现代共识协议的前身；专业工程师必须掌握它，才能理解分布式数据库、配置中心、元数据服务背后的一致性保证，而不是停留在 API 层面。

### 2. 底层原理剖析
Paxos 中每个 Acceptor 维护两个变量：last_prepared（它响应过的最高提案编号）和 accepted_value（已接受的值，若没有则为 null），以及对应的 accepted_proposal_id。Prepare(n) 处理规则：if n > last_prepared then last_prepared = n; return Promise(n, accepted_proposal_id, accepted_value) else return Reject。注意：一旦 Acceptor Promise 了编号 n，它就不能再接受任何编号小于 n 的提案，直到它收到更高编号的 Prepare。这是通过将 last_prepared 更新为 n 实现的。\n多数派重叠证明：设集合大小为 N，多数派即大小 > N/2 的任意子集。对任意两个多数派 A 和 B，|A ∩ B| = |A| + |B| - |A ∪ B| ≥ |A| + |B| - N > N/2 + N/2 - N = 0，故交集非空。这意味着：如果一个值 v 在一个编号为 n 的提案中被 Acceptor 集合 A 接受（即形成多数派），那么任何后续的 Prepare 编号 m（m > n）的请求发送给一个多数派 B 时，B 中至少有一个 Acceptor 同时属于 A，且该 Acceptor 的 accepted_proposal_id 至少是 n（因为它在编号 n 时接受了 v）。因此 Prepare(m) 会从该 Acceptor 处得到 accepted_value = v，从而 Proposer 在 Phase 2 中必须使用 v 作为值，不能重新选择。这个性质是归纳的：即使有多个提案交替，每个新提案在 Prepare 阶段都会发现之前所有已选定的值，从而保证一致性。\n对比前端概念：Paxos 的 Prepare/Promise 类似于前端中的“乐观锁版本号”（比如数据库的 CAS 操作）——Prepare 的编号就是版本号，Acceptor 通过版本号判断是否接受。但更本质的对比是：它不同于 TypeScript 的接口——接口是编译期约束，而 Paxos 是运行期协议；也不同于 JavaScript 的事件循环——事件循环是单线程调度，Paxos 是跨节点状态同步。Paxos 的多数派重叠类似于前端中的“投票算法”（如 Redux 的 reducer？不，那是纯函数），更准确的类比是：当多个副本需要同步时，不能用前端熟悉的“回调”或“事件”模式，因为网络不可靠，必须用法定人数（quorum）来抽象。

### 3. 基础代码与实战验证
```text
# Acceptor 状态\nclass Acceptor:\n    last_prepared = 0\n    accepted_proposal_id = None\n    accepted_value = None\n\n    def prepare(self, n):\n        # 若新提案编号大于已响应过的最大编号，则更新并承诺\n        if n > self.last_prepared:\n            self.last_prepared = n\n            # 返回当前已接受的最大编号及其值（可能为 None）\n            return Promise(n, self.accepted_proposal_id, self.accepted_value)\n        else:\n            return Reject(n)\n\n    def accept(self, n, value):\n        # 只有编号不小于已承诺的最大编号时才接受\n        if n >= self.last_prepared:\n            self.last_prepared = n\n            self.accepted_proposal_id = n\n            self.accepted_value = value\n            return Accepted(n, value)\n        else:\n            return Reject(n)\n\n# Proposer 逻辑\ndef propose(value):\n    # Phase 1: Prepare\n    n = generate_new_proposal_id()  # 必须唯一且递增\n    promises = send_prepare_to_majority(n)\n    if len(promises) < majority_size():\n        return "abort"\n    # 从所有 Promise 中取出 accepted_proposal_id 最大的已接受值\n    max_id = -1\n    chosen_value = None\n    for promise in promises:\n        if promise.accepted_proposal_id is not None and promise.accepted_proposal_id > max_id:\n            max_id = promise.accepted_proposal_id\n            chosen_value = promise.accepted_value\n    # 如果存在已接受值，则必须使用该值；否则使用自由值\n    if chosen_value is not None:\n        value = chosen_value\n    # Phase 2: Accept\n    responses = send_accept_to_majority(n, value)\n    if len(responses) >= majority_size():\n        return "chosen: " + value\n    else:\n        return "abort"\n\n关键注释：Acceptor 在 accept 中检查 n >= last_prepared 是因为如果它已经 Promise 过更大的编号，则不能接受；而等于的情况是允许的（在收到 Prepare 后，立即 Accept 同一个编号）。Proposer 在 Phase 1 之后，必须从所有返回的 Promise 中提取 accepted_proposal_id 最大的那个 value，这是保证安全性的关键——因为多数派重叠，这个值可能来自之前已经选定的值，若忽略它而自行选值，就会破坏一致性。
```

### 4. 常见误区与进阶思考
常见误区 1：认为 Prepare 阶段是为了选出一个值。实际上 Prepare 只是锁定编号并收集信息，值的选择在 Accept 阶段，且值必须取自 Prepare 阶段返回的最大已接受提案（如果存在）。如果 Proposer 在 Prepare 后自由选择新值，当并发 Prepare 发生时，会导致两个不同的值被不同的多数派接受，但 Paxos 通过“必须使用最大编号提案的值”规则避免这一点。\n常见误区 2：认为只要 Prepare 阶段获得多数派 Promise，就一定能 Accept 成功。实际上 Promise 只是一个承诺，后续更高编号的 Prepare 可能使 Acceptor 拒绝 Accept。所以 Paxos 可能被更高编号的提案“打断”，需要重新发起 Prepare。但这不影响安全性，只影响活性。\n深度思考题：考虑两个 Proposer P1 和 P2 并发运行：P1 发送 Prepare(1) 获得多数派 Promise，然后 P2 发送 Prepare(2) 也获得多数派 Promise（可能包括相同或不同的 Acceptor），接着 P1 发送 Accept(1, v1) 会被这些 Acceptor 拒绝（因为 last_prepared=2），P2 发送 Accept(2, v2) 可能成功。请说明为什么即使 P1 的 Prepare 先于 P2 的 Prepare，P2 也必须检查 P1 的 Prepare 是否已导致某个值被接受？如果 P1 在 Prepare 后没有发送 Accept，那么 P2 的 Prepare 不会看到任何已接受值，此时 P2 可以自由选择 v2，这是否会违反一致性？请结合多数派重叠和“已接受值”的定义分析。答案：如果 P1 没有 Accept，那么没有任何值被选定，P2 自由选择 v2 不会违反一致性，因为不存在“之前选定的值”。如果 P1 已经 Accept 并形成多数派，那么 P2 的 Prepare(2) 发送给多数派时，由于两个多数派交集，至少有一个 Acceptor 接受了 v1（且其 accepted_proposal_id=1），P2 会从该 Acceptor 处得到 v1，因此 P2 必须选择 v1。所以安全性成立。这验证了 Prepare 阶段返回已接受值的作用。
