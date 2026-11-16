---
title: "每日基础技术总结 · 2026-11-17 · Elasticsearch Shard级别的分片副本同步与延迟监控"
date: 2026-11-17 07:02:36
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-17 · Elasticsearch Shard级别的分片副本同步与延迟监控

## 📚 今日主题

> **Elasticsearch Shard级别的分片副本同步与延迟监控**（后端基础）

### 1. 核心概念速览
Elasticsearch 的副本同步发生在 shard 级别：每个主分片（primary shard）拥有一个复制组，由主副本和若干处于 in-sync set 中的副本（replica shard）共同推进同一个操作序列。客户端写入先路由到 primary；primary 为该操作分配 primary_term 与 seq_no，写入 Lucene 与 translog，再把操作转发给所有当前 in-sync allocation id；只有这些 in-sync 副本完成操作并返回确认后，primary 才向客户端返回成功。这里的机制不是最终一致性的异步广播，而是单主、顺序号驱动、按 shard 复制的状态机同步。副本在收到操作后独立写 translog，再按各自的 refresh/flush 周期使数据对搜索可见并持久化，因此“写入成功”保证的是复制组 ack 与 durability，不保证所有副本在同一毫秒可被查询；延迟监控就是观察复制组内各 shard 的 seq_no/local_checkpoint/global_checkpoint、translog、refresh/flush、allocation state 与 indexing queue 之间的偏差。它在计算机体系中的位置：位于存储引擎、网络复制协议与集群协调之间；它是 ES 高可用和写一致性的最小执行单元，也是性能退化、容量倾斜、故障恢复的第一现场。专业工程师必须掌握它，因为 shard 副本的同步语义决定了写入延迟、读扩展、failover 风险、恢复时间与监控告警的有效边界。

### 2. 底层原理剖析
底层机制可以按 shard 复制状态机描述：1. 写入路由：coordinator 根据 routing 选择 index shard 的 primary，cluster state 决定该 shard 当前有哪些 allocation id 属于 in-sync set。2. 操作定序：primary 在写入前递增 seq_no，并附带 primary_term 防止脑裂旧 primary 覆盖新操作；真实 ES 还会校验 active shards 数是否满足 wait_for_active_shards。3. 双阶段推进：primary 先 apply 到本地 Lucene/translog，再向所有 in-sync replicas 发送同一操作；每个 replica 必须按连续序号 apply 并更新 local_checkpoint，缺号则触发从 primary translog 补日志或进入 recovery。4. 确认屏障：当目标 in-sync copies 全部确认，primary 将 global_checkpoint 推进为所有 in-sync copies local_checkpoint 的最小值，并允许 translog 按保留水位裁剪。5. 搜索可见性：refresh 是 shard 本地行为，primary 与 replica 可不同步 refresh，因此 durable ack 不等于 search visible；flush/fsync 负责 translog 与 Lucene commit 的持久化，merge/IO/refresh 竞争会放大可见延迟。6. 延迟类型：sync lag = primary.max_seq_no 或 local_checkpoint 与 replica.local_checkpoint 的差；visible lag = replica.local_checkpoint 与 replica 当前 refresh 水位的差；recovery lag = initializing/corrupted/stale allocation id 尚未进入 in-sync；queue lag = indexing buffer/throttle/merge pressure 造成操作排队。7. 监控位置：cluster state/routing 判断 in-sync 与 allocation 健康，cluster health 判断 active/unassigned shard，per shard stats 与 nodes stats 判断 refresh/flush/indexing 压力，tasks/indexing pressure 判断写请求是否卡在队列。与前端已有概念对比：Java interface 是 JVM 运行期强约束的显式契约，实现方必须声明；TypeScript interface 是编译期结构化类型，运行期擦除。ES shard replica sync 的契约既不是 Java 显式声明，也不是 TS 类型擦除，而是运行期协议契约：primary_term/seq_no/in-sync allocation ids/global_checkpoint 是必须满足的状态收敛条件；它校验的是分布式日志序号与副本 ack 集合，而非类型兼容。相同点是三者都定义边界条件，不同点是 ES 的边界由网络、持久化、集群状态共同强制执行，失败会改变拓扑与一致性水位。

### 3. 基础代码与实战验证
```text
class Shard:
    def __init__(self, role, shard_id):
        self.role = role            # primary 或 replica；对应同一 index shard 的复制组角色
        self.shard_id = shard_id    # allocation id；cluster state 用它标识 in-sync copy
        self.max_seq_no = 0         # 已分配的最大操作序号；真实 ES 中还有 primary_term
        self.local_checkpoint = 0   # 本分片已连续 apply 的最大 seq_no
        self.applied = 0            # 已写入 Lucene/translog 的操作数
        self.visible = 0            # refresh 后对搜索可见的操作数
        self.failed = False

    def apply(self, seq):
        # 副本收到操作后必须按序号连续 apply；乱序或断号会触发从 primary translog 补日志
        self.max_seq_no = seq
        self.applied = seq
        self.local_checkpoint = seq

class Primary:
    def __init__(self):
        self.replicas = []
        self.in_sync = set()
        self.global_checkpoint = 0

    def add_replica(self, r):
        self.replicas.append(r)
        self.in_sync.add(r.shard_id) # 新副本恢复完成前不会进入 in-sync set

    def write(self, doc, fail_id=None):
        # 1. coordinator 将请求路由到 primary，primary 分配 seq_no
        self.max_seq_no += 1
        seq = self.max_seq_no
        self.apply(seq)

        # 2. 把同一操作转发给所有当前 in-sync allocation id
        target = {r.shard_id for r in self.replicas if r.shard_id in self.in_sync}
        acks = set()
        for r in self.replicas:
            if r.shard_id not in target:
                continue
            if r.failed or r.shard_id == fail_id:
                # 副本失败后移出 in-sync set；cluster 会把它标记为失败/初始化/未分配并重新复制
                self.in_sync.discard(r.shard_id)
            else:
                r.apply(seq)
                acks.add(r.shard_id)

        # 3. 只有目标 in-sync copies 全部确认，primary 才返回成功
        if acks == target:
            # 4. global checkpoint 取 in-sync copies local checkpoint 的最小值
            cp = [r.local_checkpoint for r in self.replicas if r.shard_id in target]
            self.global_checkpoint = min(cp) if cp else seq
            return True
        return False

def visible_refresh(r, seq):
    # refresh 是搜索可见性边界；真实 ES 不要求所有副本在同一 seq_no 同步 refresh
    r.visible = seq

def lag(p, r):
    sync_lag = p.local_checkpoint - r.local_checkpoint          # 操作推进滞后
    visible_lag = r.local_checkpoint - r.visible                # 引擎应用后尚未 refresh 的搜索滞后
    return sync_lag, visible_lag

p = Primary()
r1 = Shard('replica', 'r1')
r2 = Shard('replica', 'r2')
p.add_replica(r1)
p.add_replica(r2)

for i in range(1, 4):
    p.write('doc' + str(i))

visible_refresh(r1, p.applied) # r1 已 refresh，可被查询到
visible_refresh(r2, p.applied - 1) # r2 仍落后一个文档

# 模拟 r2 在后续写入中失败：只要 r1 仍属于 in-sync set，写请求仍可成功
ok = p.write('doc4', fail_id='r2')
print(ok, lag(p, r1), lag(p, r2))

# 实战对应 ES API 信号：
# _cluster/state 看 routing_table 与 allocation ids
# _cluster/health 看 active_shards_percent_as_number、unassigned_shards
# _stats/refresh 与 _stats/translog 看 refresh/flush 与 translog 积压
# _nodes/stats 看 indexing throttle、indexing buffer、merge pressure
# 真实 ES 还会把 local/global checkpoint 与 max_seq_no 作为更精确的同步水位，按版本在 stats 或内部 telemetry 暴露
```

### 4. 常见误区与进阶思考
常见误区1：把写入返回成功理解为所有副本都已完成搜索可见。真实 ES 只保证当前 in-sync copies 已 ack 并进入各自 translog/Lucene 路径；副本的 refresh 是异步本地事件，查询可能读到 primary 但读不到 replica，尤其在 replica 刚完成恢复、refresh 被节流或请求命中不同分片时。误区2：把 number_of_replicas、unassigned_shards=0 或 _cat/shards 的 docs count 等同于副本同步延迟。number_of_replicas 只是期望配置；in-sync set 才是写入确认集合。unassigned_shards=0 只表示分片被放置，不表示每个活跃副本的 seq_no 已追上 global checkpoint。docs count 受 refresh/flush 与删除操作影响，无法区分 durable 应用滞后与 search 可见滞后。进阶思考题：如果监控发现 primary.local_checkpoint=100，某个仍被标记为 in-sync 的 replica.local_checkpoint=95，而 global_checkpoint=95，写入为何仍可能返回成功？这暴露了 ack 屏障、in-sync membership 与 checkpoint 计算之间哪一层可能不一致？为什么单纯把副本移出 in-sync set 并不是最终修复，而必须恢复该副本的连续 seq_no、重新进入 in-sync 并推进 global_checkpoint 才能恢复完整容错度？
