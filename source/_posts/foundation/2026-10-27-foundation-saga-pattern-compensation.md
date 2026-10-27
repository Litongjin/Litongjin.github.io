---
title: "每日基础技术总结 · 2026-10-27 · 分布式事务Saga模式的最终一致性补偿机制设计"
date: 2026-10-27 11:29:41
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-27 · 分布式事务Saga模式的最终一致性补偿机制设计

## 📚 今日主题

> **分布式事务Saga模式的最终一致性补偿机制设计**（后端基础）

### 1. 核心概念速览
定义：Saga 是面向长活事务（Long-Lived Transaction）的分布式事务模型——把一个跨越 N 个服务边界的全局事务拆成 N 个本地事务 T1…Tn，并为每个 Ti 定义补偿动作 Ci，Ci 在业务语义上是 Ti 的逆操作。正常路径按序推进；第 k 步失败时，对已完成前缀按 T(k-1)…T1 逆序执行补偿，使全局收敛到「全部生效」或「全部被抵消后的等价状态」。

本质：用「正向动作 + 逆动作 + 可持久化可重试的状态机」把分布式原子提交降级为最终一致性。被放弃的是隔离性（中间态对外可见）与原子性（不存在瞬时全有或全无）；换来的是无长事务锁、无协调者阻塞提交、可覆盖异构参与者（HTTP / RPC / MQ / 第三方支付）。

它解决的问题：2PC/XA 要求所有参与者实现事务协议、协调者持锁至提交、故障时阻塞、跨第三方系统不可用。Saga 只要求参与者提供「本地事务 + 补偿接口」，是少数能覆盖非事务型外部依赖的通用方案。

体系位置：一致性方案光谱的一环——强一致（可串行化、2PC、Raft/Paxos）→ TCC（Try 预留资源换取隔离性）→ Saga（补偿换取可用性）→ 事务性发件箱 / 本地消息表（可靠事件投递）→ 最大努力通知。它同时是工作流引擎（Temporal、Cadence、Seata Saga）与事件驱动架构的底层骨架。

为什么必须掌握：任何跨边界写操作都必然遭遇部分失败。Saga 给出了这一问题的通用答案——记录状态、可重试地执行逆操作、把不可自动恢复的部分显式交给人工对账。它也是理解 TCC、发件箱模式、事件溯源，以及编排 / 协同两种拓扑的共同底座。

### 2. 底层原理剖析
一、运行机制（不变量层）
1) 状态持久化是前提：正向业务写入与其状态记录（saga_log）必须落在同一个本地事务里提交。否则进程崩溃后无法判定该步是否已生效，补偿变成猜测。这是事务性发件箱的同源思想。
2) 执行器是持久化状态机：重启后从 saga_log 恢复，依据已完成前缀决定继续正向还是进入补偿，因此进度可恢复、可审计。
3) 幂等：T 与 C 都只保证 at-least-once。用业务唯一键 + 幂等去重表（或唯一索引）把 at-least-once 收敛为 exactly-once 效果。补偿的幂等键必须持久化，且不能被正向步骤的键覆盖。
4) 空补偿与悬挂：C 先到（T 的请求超时后被延迟重发）产生空补偿；T 后到产生悬挂。解法是唯一键占位 + 状态机单向迁移（INIT→DONE→COMPENSATED，其它迁移一律拒绝），使先后顺序与重复投递都不影响结果。
5) 补偿必须最终可达：重试采用指数退避，超阈值进入 FAILED_MANUAL（死信 + 人工工单）。系统终态因此有三类：COMPLETED / COMPENSATED / FAILED_MANUAL。
6) 无隔离性的后果：中间态可被并发事务读到，产生脏读、丢失更新、不可重复读。缓解手段是语义锁（把中间态标记为 SUSPENDED，并发读写直接拒绝或等待）、版本号 / 条件更新（UPDATE … WHERE version = ?）、可交换更新（加减法），以及 Try 阶段预留资源（此时退化为 TCC）。

二、执行器伪代码
状态迁移：INIT → DONE → COMPENSATED；INIT → FAILED

forward(i):
  if log[i].state != INIT: return            # 状态机拒绝非法迁移 → 幂等，兼防悬挂
  TX { apply(Ti, payload); log[i] = {state: DONE, comp: extract(payload)} }

compensate(k):
  for j = k down to 0:
    if log[j].state != DONE: continue        # 未执行（空补偿）或已补偿（重复补偿）跳过
    retry with backoff:
      TX { apply(Cj, log[j].comp); log[j].state = COMPENSATED }
    if 重试耗尽: log[j].state = FAILED_MANUAL; trigger(对账 + 人工工单)

两种拓扑：编排（orchestration，集中式协调器持有状态机，易观测、易统一超时与补偿策略，但协调器成为逻辑中心）与协同（choreography，各服务订阅事件自行推进，解耦但全局状态难观测、易出环形依赖）。本质差别只是状态机集中在一处还是分散在各服务本地状态中。

三、与前端已有概念的对照
- Redux 的 undo/redo 中间件为每个 action 生成 inverse action，结构与「正向 + 补偿」同构；但 Redux 是单线程内存中的确定性重放，undo 不会失败、不会重复、不存在并发观察者。Saga 的补偿是网络调用，会超时、乱序、重复，且中间态已被其他事务观察到，因此必须叠加幂等、状态机与隔离性设计。
- 前端乐观更新（optimistic UI）失败时回滚本地状态，成本为零且不撤销服务端已产生的副作用；Saga 的补偿是一次新的业务写入，副作用真实且可能无法在语义上求逆（已发出的短信、已被消费的库存）。
- 与「接口」类比的边界：TS / Java 的接口是编译期静态类型约束，违反即编译失败；Saga 的补偿契约是运行期、跨进程、可被静默违反的时序与状态迁移约束，只能靠幂等键、单向状态机与对账兜底。

### 3. 基础代码与实战验证
```text
# 极简 Saga 执行器：无框架，内存模拟两个服务的本地库 + Saga 日志 + 幂等去重表
db = {
    'acct': {'u1': 100},      # 账户服务本地库
    'points': {'u1': 0},      # 积分服务本地库
    'saga_log': {},           # (saga_id, step) -> {'state': INIT|DONE|COMPENSATED|FAILED, 'comp': 补偿上下文}
    'dedup': set(),           # 幂等去重键 (saga_id, step, 'C')，必须持久化，随进程重启仍在
}

def debit(p):   db['acct']['u1'] -= p      # T1 正向：扣款
def refund(p):  db['acct']['u1'] += p      # C1 补偿：退款（新的正向写入，不是存储引擎的 undo）
def credit(p):  db['points']['u1'] += p    # T2 正向：发积分

def do_step(saga_id, step, forward, payload):
    log = db['saga_log'].setdefault((saga_id, step), {'state': 'INIT', 'comp': None})
    if log['state'] != 'INIT':
        return log['state']          # 状态机只允许 INIT→DONE；重复到达的正向请求被拒 → 幂等 + 防悬挂
    forward(payload)                 # 业务写入
    log['state'] = 'DONE'            # 关键：与业务写入同一本地事务提交，崩溃后据此判定是否需要补偿
    log['comp'] = payload            # 补偿只依赖持久化上下文，不对业务数据做二次查询（避免脏读）
    return log['state']

def compensate_from(saga_id, last_step):
    for step in range(last_step, -1, -1):        # 逆序：补偿是前缀回退，顺序不可乱
        log = db['saga_log'].get((saga_id, step))
        if not log or log['state'] != 'DONE':
            continue                 # 只补偿 DONE：未执行（空补偿）或已补偿（重复补偿）直接跳过
        key = (saga_id, step, 'C')
        if key in db['dedup']:
            continue                 # 幂等门闩：响应丢失后的重发不会二次退款
        refund(log['comp'])          # 执行逆操作
        db['dedup'].add(key)         # 先落幂等键，再置状态；两者同事务可进一步收紧
        log['state'] = 'COMPENSATED' # DONE→COMPENSATED 单向迁移，此后到达的 T 一律拒绝

def transfer(saga_id, amount, fail_at=None):
    try:
        do_step(saga_id, 0, debit, amount)
        if fail_at == 1:
            raise RuntimeError('points service unavailable')   # 模拟 T2 执行前的服务 / 网络故障
        do_step(saga_id, 1, credit, amount)
        return 'COMPLETED'
    except Exception:
        compensate_from(saga_id, 1)  # 补偿已完成前缀（此处仅 step 0）
        return 'COMPENSATED'

print(transfer('s1', 30))             # 正常路径：余额 70，积分 30
print(transfer('s2', 10, fail_at=1))  # T1 生效后失败 → 触发补偿退款
print(db['acct'], db['points'])       # 余额 70，积分 30：s2 的扣款被抵消，全局收敛到一致状态
print(compensate_from('s2', 1))       # 再次补偿：dedup 命中，余额不变 → 幂等性验证
```

### 4. 常见误区与进阶思考
误区一：把 Saga 的补偿等同于数据库回滚。回滚是存储引擎内部的 undo：物理恢复旧值、原子、隔离、不会失败、不对外可见；补偿是一条新的正向业务操作，语义上求逆、可能失败、可能不满足可交换性，且无法撤销已被外部世界观察到的效果（已发短信、已被下游消费的消息）。因此补偿不能实现为「先读旧值再写回」的 UPDATE accounts SET balance = 100 WHERE id = 1——并发下它会覆盖其他事务的写入；必须使用条件原子更新（SET balance = balance + ?）或版本号校验。

误区二：认为只要补偿逻辑写对就等价于强一致，从而忽略隔离性缺失带来的业务异常。Saga 不保证读已提交，更不保证可串行化：中间态对外可见，基于中间态做出的外部决策不可补偿。加长重试、加大版本号、提高数据库隔离级别都无法解决这类「不可补偿的读」，只能在架构层规避（语义锁让中间态不可读、Try 阶段预留资源、把决策改为最终一致的两阶段确认）。

思考题：构造一个执行序，使「账户扣款 T1 → 发放积分 T2」这条 Saga 在其补偿完成之后，另一笔交易已经基于 T1 之后的中间态余额发放了不可回收的优惠券。请论证：为什么这种不可补偿的读无法通过增加重试次数、调整隔离级别参数或引入版本号自动解决？并给出至少两种架构层面的规避方案，说明各自在吞吐、时延与实现复杂度上的代价。
