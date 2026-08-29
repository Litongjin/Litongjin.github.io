---
title: "每日基础技术总结 · 2026-08-30 · 事务的 ACID 特性"
date: 2026-08-30 06:57:23
categories: [技术分享]
tags: ["技术分享", "后端基础（Node.js / Java / Python）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-30 · 事务的 ACID 特性

## 📚 今日主题

> **事务的 ACID 特性**（后端基础（Node.js / Java / Python））

### 1. 核心概念速览
事务是数据库管理系统（或具备持久化状态的存储引擎）将一组读写操作封装为单个逻辑执行单元，并保证该单元在并发与故障场景下具有原子性（Atomicity）、一致性（Consistency）、隔离性（Isolation）、持久性（Durability）的机制。其本质是对状态迁移的约束：事务要么整体提交生效，要么整体回滚如未发生；事务执行前和执行后都必须满足既定的完整性约束（一致性）；多个事务并发执行的效果必须等价于某个串行顺序执行的效果（隔离性）；一旦提交，事务的修改必须永久保留，即使系统崩溃。ACID 解决的核心问题是：在多步状态变更和并发访问中，如何确保数据的正确性、可恢复性和可预测性。它在整个计算机体系中位于存储与分布式系统的核心抽象层，是数据库、消息队列、对象存储、分布式事务等一切有状态系统的基石。专业工程师必须掌握 ACID，是因为所有业务系统的数据正确性最终都依赖于底层事务语义；不理解 ACID，就无法评估框架、中间件或数据库在异常场景下的真实行为，也无法设计出可恢复、可审计、可并发的数据系统。前端工程师往往只面对无共享状态的 UI 渲染与即时交互，资源竞争和崩溃恢复由浏览器和服务器代管，一旦转向后端与 AI 数据流水线，ACID 便成为必须亲自理解和保障的底线。

### 2. 底层原理剖析
ACID 的底层机制可以从日志（Write-Ahead Logging, WAL）、锁/多版本并发控制（MVCC）、约束检查与恢复协议三个维度拆解。

1. 原子性的实现：事务的所有操作被记录到事务日志中，采用 undo 日志记录旧值。执行过程中，数据页先被修改到内存缓冲区，但必须同步或异步持久化日志。若事务在提交前失败，恢复进程根据 undo 日志将数据页回滚到事务开始前状态；若在提交后崩溃，则根据 redo 日志重放已提交但未刷盘的操作。原子性本质是‘日志先于数据落盘’的顺序保证，配合崩溃恢复的幂等重放/回滚，实现‘全有或全无’。

2. 一致性的实现：一致性不是数据库主动执行的单一机制，而是应用逻辑、完整性约束（主键、外键、唯一约束、检查约束、触发器）与原子性、隔离性共同作用的结果。事务开始前数据库处于满足约束的状态；事务内部的任意中间状态可违反约束，但提交后必须重新满足所有约束。数据库通过约束引擎在每个语句或提交时校验，同时依赖原子性保证回滚不会产生半更新，依赖隔离性保证其他事务看不到中间不一致状态。因此一致性是 ACID 中唯一由语义层定义的目标，其他三个是达成该目标的手段。

3. 隔离性的实现：主流数据库通过两阶段锁（2PL）或 MVCC 实现。2PL 下，读写均需获取锁；锁分为共享锁与排他锁；事务分为加锁阶段与释放锁阶段，释放后不再加锁，从而保证可串行化。MVCC 下，每个事务看到的是某个快照版本，写操作生成新版本，读不阻塞写、写不阻塞读；通过提交版本号与活跃事务列表判断可见性，隔离级别的强弱取决于快照获取时机与写冲突检测策略。隔离级别（读未提交、读已提交、可重复读、可串行化）本质上是放宽了串行化等价条件，换取并发度。

4. 持久性的实现：依赖 redo 日志与刷盘策略。事务提交时，必须将 redo 日志写入稳定存储（fsync），数据页本身可延后刷盘；崩溃后根据 redo 日志重放，保证已提交事务不丢失。组提交（group commit）通过合并多个事务的 fsync 提高吞吐，牺牲单事务延迟换取批量持久化。

与前端已有概念对比：数据库事务的‘原子性’不同于前端 JS 引擎的事件循环——JS 的同步代码块天然不可被中断，但若抛出异常，之前修改的 DOM 或内存对象不会自动回滚；数据库通过日志实现显式回滚，而 JS 无状态回滚机制。‘隔离性’类似前端并发控制中的竞态处理：浏览器中同一事件循环内的代码无共享内存并发，但跨线程的 SharedArrayBuffer 需要锁或原子操作，其底层内存模型（weakly ordered）与数据库的并发控制同源。‘持久性’与前端 localStorage/IndexedDB 的刷盘保证不同：浏览器的持久化不提供 fsync 级别的崩溃一致性，而数据库 redo 日志是显式的日志协议。‘一致性’则类比 React 的不可变数据与协调过程——React 的 render 阶段可以被打断（中间态），但 commit 阶段必须一次性生效，这与事务的中间状态隐藏和提交点概念在本质上是一致的。

### 3. 基础代码与实战验证
以下用 Python 纯标准库模拟一个最小事务引擎的核心机制，展示日志、回滚和提交点。不依赖任何框架，聚焦底层流程。

```python
class TransactionRecord:
    def __init__(self, tid):
        self.tid = tid
        self.undo_log = []  # 每个元素为 (key, old_value)
        self.committed = False

class MiniDB:
    def __init__(self):
        self.data = {}           # 主数据区，仅反映已提交结果
        self.active_tx = None    # 当前活动事务（单线程简化）
        self.redo_log = []       # 持久化重做日志，模拟 WAL

    def begin(self):
        if self.active_tx is not None:
            raise Exception("并发不支持，同一时间仅一个事务")
        self.active_tx = TransactionRecord(len(self.redo_log) + 1)
        return self.active_tx

    def read(self, key):
        # 事务内读操作可见自己的写（通过 undo_log 中的旧值推导），但这里直接读主数据区
        # 简化：默认读已提交数据，且事务写后立即更新主数据区并记录 old value。
        return self.data.get(key)

    def write(self, key, value):
        tx = self.active_tx
        if tx is None:
            raise Exception("没有事务")
        old_value = self.data.get(key)  # 记录旧值，供回滚使用
        tx.undo_log.append((key, old_value))
        self.data[key] = value          # 就地修改，但尚未提交

    def commit(self):
        tx = self.active_tx
        # 持久化：先将日志写入 redo（这里用列表追加模拟 fsync 稳定存储）
        # 日志内容包含事务 ID 及所有新值，供崩溃后重放
        self.redo_log.append(('COMMIT', tx.tid, dict(self.data)))
        tx.committed = True
        self.active_tx = None
        # 正式提交点：此后事务的修改不可撤销

    def rollback(self):
        tx = self.active_tx
        # 按 undo 日志逆序恢复旧值
        for key, old_value in reversed(tx.undo_log):
            if old_value is None:
                del self.data[key]
            else:
                self.data[key] = old_value
        self.active_tx = None

# 验证原子性：执行一半失败，回滚后数据恢复
db = MiniDB()
db.begin()
db.write('balance', 100)
db.write('history', 'transfer')
# 模拟中途异常导致回滚
db.rollback()
assert db.data == {}, "回滚后主数据区应为空"

# 验证持久性：提交后 redo 日志存在，即使主数据区丢失也可恢复
db.begin()
db.write('balance', 200)
db.commit()
assert db.redo_log[-1][0] == 'COMMIT'
# 崩溃模拟：主数据区清空，用 redo_log 恢复
recovered = db.redo_log[-1][2]
assert recovered['balance'] == 200
```

关键行注释：
- `self.undo_log.append((key, old_value))`：在修改前记录旧值，这是原子回滚的前提；数据库用 undo 日志存页级旧版本。
- `self.redo_log.append(('COMMIT', tx.tid, dict(self.data)))`：模拟 WAL——先持久化日志再确认提交；崩溃恢复时以日志为准。
- `for key, old_value in reversed(tx.undo_log)`：逆序回滚，保证依赖关系正确的状态还原。

真实数据库比这复杂，但本质骨架如此：日志即真相，数据页是日志的缓存；原子性依赖 undo，持久性依赖 redo，两者都是围绕日志的顺序写展开。

### 4. 常见误区与进阶思考
认知误区一：将 ACID 中的‘一致性’理解为数据库自动保证的某个具体值（如总和不变或某个业务恒等式）。实际上，数据库只保证约束约束强制的不变量，业务级一致性（如转账前后总额相等）完全由应用在事务内显式维护，数据库无法感知。若事务只做了 A 减 100 而忘记 B 加 100，提交后数据库依然成功，因为约束并未违反。很多工程师把业务 bug 归咎于‘事务没生效’，根源是混淆了数据库一致性（约束满足）与业务一致性（语义完整）。

认知误区二：以为事务隔离级别越强越好，盲目使用可串行化。隔离性与性能是本质矛盾：可串行化需要更强的锁或冲突检测，会放大等待和回滚；实际业务往往只需读已提交或可重复读，通过合理设计（如乐观锁、唯一约束、幂等键）即可在更低隔离级别下维持正确性。专业工程师需理解每个隔离级别允许的异常（脏读、不可重复读、幻读），并针对具体业务选择最弱可用级别，而不是把更高隔离级别当安全默认值。

进阶思考题：在 MVCC 的读已提交隔离级别下，如果一个事务先读数据 A（快照 V1），随后另一个事务提交了对 A 的更新（生成 V2），那么第一个事务在之后的语句中再读 A，读到的可能是 V1 还是 V2？请结合快照生成时机与可见性判定解释。更本质地，若我们撤掉 MVCC 而只用单版本加锁，如何在不损失‘读不阻塞写’的前提下实现同样的隔离级别——这暴露了多版本技术存在的根本原因，也是深入理解隔离性的试金石。
