---
title: "每日基础技术总结 · 2026-07-06 · kube-proxy 的 iptables 与 IPVS 模式差异及调度算法"
date: 2026-07-06 08:00:00
categories: [技术分享]
tags: ["技术分享", "DevOps 与云原生"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-06 · kube-proxy 的 iptables 与 IPVS 模式差异及调度算法

## 📚 今日主题

> **kube-proxy 的 iptables 与 IPVS 模式差异及调度算法**（DevOps 与云原生）

### 1. 核心概念速览
kube-proxy 是 Kubernetes 集群中负责实现 Service 抽象的网络组件，其本质是一个位于每个 Node 上的用户态控制器，通过 watch API Server 中 Service 与 EndpointSlice 的变化，将 Service 的虚拟 IP（ClusterIP）映射到一组真实的 Pod IP:Port。iptables 模式利用 Linux 内核 Netfilter 框架的 iptables 工具，为每个 Service 生成一系列 PREROUTING/OUTPUT 链规则，通过 DNAT 将目标为 ClusterIP 的包随机（默认随机端口匹配）改写为某个 Endpoint 的 IP:Port，并利用 conntrack 保证回程流量正确还原。IPVS 模式则利用内核中的 IP Virtual Server 模块，在内核态建立一个虚拟服务器表，通过调度算法（如 rr, wrr, lc, sh 等）选择后端，相比 iptables 的线性规则遍历，IPVS 使用哈希表查找，复杂度为 O(1)，在大规模 Service 场景下性能更优、规则更新更高效。该知识点属于云原生网络数据面核心机制，决定了 Service 的转发延迟、扩展性以及运维排障路径，专业工程师必须理解两种模式的底层差异才能在集群规模增长时做出正确的架构决策。

### 2. 底层原理剖析
iptables 模式的核心机制是链式规则匹配。kube-proxy 为每个 Service 创建一条 KUBE-SVC-XXX 链，并在 PREROUTING 和 OUTPUT 链上添加跳转规则，使所有目标为 ClusterIP 的包进入对应链。链中按 1/概率 的方式插入多条 DNAT 规则（例如每个 Endpoint 一条规则，每条规则有相同的匹配概率权重），命中后跳转到 KUBE-SEP-XXX 链执行 DNAT 并记录 conntrack 条目。内核在收到数据包时，按顺序逐条比对规则，直到匹配。当 Service 有 N 个 Endpoint 时，规则数量随 N 线性增长，且更新规则时 iptables 需要整体替换表，存在锁竞争和延迟。IPVS 模式则基于 Netfilter 的 hook 点注册了专用钩子，用户态通过 setsockopt 或 netlink 将 Service 和 Endpoint 信息同步到内核 IPVS 表。数据包到达时，内核根据目标 IP:Port 在哈希表中直接查找对应的虚拟服务，然后由调度算法从关联的后端列表中选出目标。IPVS 支持多种调度算法：rr（轮询）、wrr（加权轮询）、lc（最少连接）、wlc（加权最少连接）、sh（源地址哈希）、dh（目标地址哈希）等。更新操作通过增量添加/删除后端实现，不需要整体刷新。与前端对比：iptables 模式类似 JavaScript 中通过 if-else 链逐一比较每个条件，规则越多耗时越长；IPVS 模式类似使用 Map 或对象直接以 key 索引到 value，无论表多大查找时间恒定。而 IPVS 的调度算法则类似负载均衡器中的策略，是独立的选路逻辑。

### 3. 基础代码与实战验证
```text
以下为伪代码描述，不依赖具体框架，验证 iptables 与 IPVS 的核心行为差异。

# 模拟 iptables 模式：线性规则匹配
def iptables_forward(packet, rules):
    for rule in rules:  # 逐个规则顺序匹配
        if rule.match(packet):
            return rule.action(packet)  # DNAT 到某个 Endpoint
    return drop(packet)  # 无规则则丢弃

# 构建规则：每个 Endpoint 一条 DNAT 规则
rules = []
for ep in endpoints:
    rules.append(Rule(
        match=lambda p, ep=ep: p.dst_ip == service.cluster_ip and p.dst_port == service.port,
        action=lambda p, ep=ep: p.dnat(ep.ip, ep.port)
    ))

# 模拟 IPVS 模式：哈希表查找 + 调度算法
class IPVSTable:
    def __init__(self):
        self.table = {}  # key=(cluster_ip, port) -> VirtualService

    def add_service(self, service, endpoints, scheduler):
        self.table[(service.cluster_ip, service.port)] = VirtualService(endpoints, scheduler)

    def forward(self, packet):
        key = (packet.dst_ip, packet.dst_port)
        vs = self.table.get(key)  # O(1) 哈希查找
        if vs is None:
            return drop(packet)
        ep = vs.scheduler.select(vs.endpoints, packet)  # 调度算法选择后端
        return packet.dnat(ep.ip, ep.port)

# 验证：当 endpoints 数量从 1 增加到 10000，iptables_forward 的耗时线性增长，IPVSTable.forward 耗时不变。
# 实际 kube-proxy 中 IPVS 使用 netlink 同步数据，iptables 使用 iptables-restore 批量更新，差异更大。
```

### 4. 常见误区与进阶思考
常见误区一：认为 IPVS 一定比 iptables 快。实际上在小规模集群（几十个 Service）下两者差异微乎其微，且 IPVS 模式依赖内核模块 ip_vs 以及相关调度算法模块，若未加载会回退或报错；同时 IPVS 不提供 iptables 那样的灵活匹配能力（如按源 IP 做复杂条件），某些高级策略仍需结合 iptables。常见误区二：混淆 ClusterIP 的负载均衡与 kube-proxy 的调度算法。IPVS 的调度算法仅作用于数据包转发时选择后端，而 Service 的 sessionAffinity（会话保持）实现机制在 IPVS 下是独立的（通过 sh 调度或 persistent 标志），在 iptables 下通过 conntrack 记录原地址，两种机制的生效层次不同。思考题：当你在一个 5000 个 Service、每个 Service 平均 100 个 Endpoint 的集群中，从 iptables 切换到 IPVS 模式后，为什么有时会发现长连接（如 TCP 连接池）被重置？请从 conntrack 表项与连接追踪失效的角度分析。
