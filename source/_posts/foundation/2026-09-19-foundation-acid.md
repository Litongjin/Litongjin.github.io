---
title: "每日基础技术总结 · 2026-09-19 · 事务的 ACID 特性"
date: 2026-09-19 07:02:03
categories: [技术分享]
tags: ["技术分享", "后端基础（Node.js / Java / Python）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-19 · 事务的 ACID 特性

## 📚 今日主题

> **事务的 ACID 特性**（后端基础（Node.js / Java / Python））

### 1. 核心概念速览
定义：事务（Transaction）是 DBMS 对一组读/写操作施加的执行单元语义，使其满足 ACID 四条可判定性质。它解决的是两个正交维度下的状态可推理性问题：并发（多个执行流的交错执行）与故障（进程崩溃、掉电、介质损坏）。

四条性质的本质：
- Atomicity（原子性）：事务的状态转移不可分割。机制上不是「不执行一半」，而是「执行一半后能精确撤销」——由 undo log（before image，前像）实现，回滚 = 对未提交事务的所有修改反向 apply 前像。
- Consistency（一致性）：事务把数据库从一个满足全部完整性约束与业务不变量的状态带到另一个满足约束的状态。它不是独立机制，而是 A + I + D 叠加约束检查（PK / FK / UNIQUE / CHECK / 触发器 / 隔离级别）后合成的结果性质——数据库只能保证被显式声明的那部分不变量。
- Isolation（隔离性）：并发执行的事务集合，其交错执行的结果等价于这些事务按某个全序串行执行（serializability）。实现分为悲观（Strict 2PL + 死锁检测）与乐观（MVCC 版本链 + Read View 可见性判定 + 写冲突检测）两条路径。
- Durability（持久性）：COMMIT 返回之后，事务效果在任意后续故障下不可撤销。机制：WAL（write-ahead logging）+ fsync；数据页可延迟落盘，但 redo log 必须先于数据页落盘；崩溃恢复时按 redo 重放、按 undo 回滚。

体系位置（自下而上）：存储引擎事务管理器（undo/redo 段、buffer pool、锁管理器、MVCC 版本链、purge 线程）→ 恢复子系统（crash recovery、checkpoint、LSN）→ SQL/协议层（BEGIN / COMMIT / ROLLBACK、autocommit、隔离级别）→ 应用层（Java @Transactional、Node 的 connection.beginTransaction/pg client、Python DB-API 的 connection.commit）→ 分布式层（2PC/XA、TCC、Saga、本地消息表 + Outbox）。

与前端知识体系的映射：前端长期不存在事务语义。JS 的 run-to-completion 只保证单个宏任务/微任务不被打断，跨 await 之后状态可被其他异步流任意修改，等价于无隔离级别；localStorage 多 key 写入无原子性。真正具备类事务语义的前端基础设施只有两类：IndexedDB 的 transaction（scope 限定 + 自动 commit/abort）与 React Fiber 的渲染提交模型（render 阶段可中断可丢弃 = undo；commit 阶段不可中断、副作用一次性 flush = 原子提交点；current/workInProgress 双树指针切换 = 提交点的持久化切换）。

为何必须掌握：所有涉及读-改-写（库存扣减、余额、幂等、状态机流转、抢占式分配）的逻辑，其正确性边界均由 ACID 与隔离级别决定。不理解 undo/redo 与 MVCC 的实际语义，就无法判断一段代码在并发与崩溃下的真实行为，也无法在锁、版本号、乐观重试、分布式事务之间做出正确取舍。

### 2. 底层原理剖析
一、一次 UPDATE 在存储引擎内部的完整路径（InnoDB 风格）

BEGIN:
  txn_id = alloc_txn_id()
  read_view = build_read_view()        # 快照读的可见性集合：当前活跃事务集合 + 上界

UPDATE acct SET balance = balance - 30 WHERE id = 1:
  1. 定位与加锁：在聚簇索引上定位记录；当前读加 X 锁（RR 下为 next-key lock），快照读不加锁
  2. 写 undo：把旧行镜像（before image）追加到 undo 段，并记下反向操作类型（INSERT 的反向是 DELETE，UPDATE 的反向是用前像覆盖）。注意：undo 页本身的修改同样要写 redo——否则崩溃后 undo 丢失，未提交事务无法回滚，A 立即失效
  3. 改内存页：在 buffer pool 中就地修改并把数据页标记为脏页，此时内存与数据文件不一致
  4. 写 redo：把后像/delta 顺序追加到 redo log buffer；提交时 fsync 到 redo 文件

COMMIT:
  5. 向 redo 写入 commit 记录——这就是逻辑上的原子提交点。单条日志的原子性由「顺序追加 + 尾记录 checksum/LSN 校验」保证
  6. redo.fsync() 返回即意味着 D 成立；数据页可在之后任意时刻刷盘
  7. 释放锁（Strict 2PL 在提交后才释放）、标记该事务的 undo 可被 purge

ROLLBACK 或崩溃恢复:
  8. 沿 undo 版本链反向 apply 前像恢复旧值；回滚动作本身也产生 redo

二、崩溃恢复的两遍扫描

recovery():
  # redo 遍：幂等重放，把已落盘日志的效果全部作用到数据页（含未提交事务的修改）
  for rec in redo_log_since_last_checkpoint:
      page.apply(rec)
  # undo 遍：只回滚崩溃时仍 active（无 commit 标记）的事务
  for txn in txns_without_commit_marker:
      for rec in reverse(txn.undo_chain):
          page.apply(rec.before_image)

  正确性依赖三点：LSN 单调递增、日志顺序追加、单条记录可校验（无法写入半条日志）。

三、隔离性的两条实现路径

A. 悲观：两阶段锁（2PL）
  growing phase 只加锁不解锁 → shrinking phase 只解锁不加锁；若所有 X 锁持到事务结束即 Strict 2PL，可避免级联回滚。串行化由「冲突操作必须被同一把锁排序」直接推出。代价：并发度下降、死锁（需 wait-for graph 检测并选择 victim 回滚）、RR 下 next-key lock 带来更多锁冲突。

B. 乐观：MVCC + 可见性判定
  每行通过隐藏列（trx_id + roll_pointer 指向 undo）构成版本链，读操作不加锁，只判断版本可见性：

  visible(version, rv):
      if version.trx_id == rv.self:      return True    # 自己未提交的修改自己可见
      if version.trx_id in rv.active_set: return False  # 该版本来自未提交事务，沿链找更旧版本
      if version.trx_id >= rv.up_limit:   return False  # 快照之后才开始的事务，不可见
      return True                                        # 已提交且早于快照，可见

  READ COMMITTED：每条语句重建 Read View（能看到最新提交）；REPEATABLE READ：整个事务复用同一 Read View（快照固定）。MVCC 只解决读-写冲突，写-写冲突仍需锁或带版本条件的更新（UPDATE ... WHERE version = n）兜底，否则出现丢失更新。

四、隔离级别与其禁止的异常

  级别                   脏读    不可重复读   幻读            写偏斜
  READ UNCOMMITTED       可能    可能        可能            可能
  READ COMMITTED         禁止    可能        可能            可能
  REPEATABLE READ(InnoDB)禁止    禁止        快照读禁止/当前读可能  可能
  SERIALIZABLE           禁止    禁止        禁止            禁止

  关键点：REPEATABLE READ 只约束「读到的版本集合」，不蕴含串行化。PostgreSQL 的 REPEATABLE READ 实为 Snapshot Isolation，允许写偏斜（经典例：两名医生各自查询值班人数均 ≥ 2 后同时请假）；只有 SI + 冲突图检测（SSI）或纯 2PL 的 SERIALIZABLE 才排除全部串行化异常。

五、与前端已有概念的精确对照

- TS 的 interface 是编译期结构约束、运行时被完全擦除；ACID 中的 C 是运行时不变量，必须在每次状态转移时被真实求值——静态可验证与动态可验证的根本差异。
- JS 的 run-to-completion 提供的是指令级原子性，不是事务级原子性：每个 await 边界、每个 I/O 回调边界都是天然的并发交错点，此处完全没有隔离。前端做乐观更新时手写的「缓存旧值 → 修改 → 失败还原」，本质就是在应用层手工实现单记录的 undo log，且通常缺少 D（刷新即丢失）。
- Redux 的 reducer 纯函数只保证可预测性；dispatch 多次仍可被中间件与异步逻辑打断，store 的最终状态并非原子提交。React 的 commit 阶段（不可中断地 flush effect 链、一次性切换 Fiber 树指针）才是真正贴近「原子提交点 + redo 落盘」的模型。
- IndexedDB 的 transaction 生命周期绑定在事件循环任务上：一个 tick 内没有继续排队新请求就自动 commit。这是「事务边界由执行上下文决定」这一数据库语义在浏览器侧的翻版——也解释了为什么在 await 之后继续使用同一个 transaction 会抛出 TransactionInactiveError。

### 3. 基础代码与实战验证
```text
import sqlite3, os

db = 'acid_demo.db'
for p in (db, db + '-wal', db + '-shm'):
    if os.path.exists(p):
        os.remove(p)

# isolation_level=None：关闭 sqlite3 模块的隐式事务包装，由代码手写 BEGIN/COMMIT。
# 这样才能观察到事务边界在协议层的真实位置（等价于 JDBC 的 setAutoCommit(false)）。
conn = sqlite3.connect(db, isolation_level=None)
conn.execute('PRAGMA journal_mode=WAL')   # WAL：修改先顺序追加到 -wal 文件并 fsync，之后异步 checkpoint 回主库

conn.execute('CREATE TABLE acct(id INTEGER PRIMARY KEY, balance INTEGER NOT NULL CHECK(balance >= 0))')
conn.execute('INSERT INTO acct(id,balance) VALUES (1,100)')
conn.execute('INSERT INTO acct(id,balance) VALUES (2,100)')

# ---------- 验证 A / C：约束违反触发回滚，undo 前像把已修改的页逐条还原 ----------
conn.execute('BEGIN')   # 分配 txn_id、建立 Read View；此后每条修改都产生 undo 前像与 redo 后像
try:
    # 写 undo(前像 100) → 脏页进 buffer pool → redo 顺序追加，三步顺序不可交换
    conn.execute('UPDATE acct SET balance = balance - 30 WHERE id = 1')
    conn.execute('UPDATE acct SET balance = balance + 30 WHERE id = 2')
    # Consistency 由 CHECK 约束在语句层拦截，而非由隔离级别保证
    conn.execute('INSERT INTO acct(id,balance) VALUES (3,-1)')
    # 写 commit 标记并 fsync redo：此处才是不可逆的原子提交点
    conn.execute('COMMIT')
except sqlite3.IntegrityError:
    # 反向 apply 第一条 UPDATE 的前像，balance 回到 100；A 因此成立
    conn.execute('ROLLBACK')
print('A/C:', conn.execute('SELECT * FROM acct ORDER BY id').fetchall())   # [(1, 100), (2, 100)]

# ---------- 验证 I：未提交版本对其它连接不可见，且事务内快照固定 ----------
conn2 = sqlite3.connect(db, isolation_level=None)
conn2.execute('BEGIN')   # SQLite 为延迟事务：第一条读语句才真正建立读快照
conn.execute('BEGIN')
conn.execute('UPDATE acct SET balance = balance - 50 WHERE id = 1')
# MVCC 快照读：沿版本链做可见性判定，跳过 trx_id 属于活跃集合的版本
print('I(未提交):', conn2.execute('SELECT balance FROM acct WHERE id = 1').fetchone())  # (100,)
conn.execute('COMMIT')   # 写 commit 标记 + fsync，新版本对后续新建的 Read View 才可见
# 同一事务复用同一 Read View，因此仍读到旧快照——这正是 REPEATABLE READ 而非 READ COMMITTED
print('I(事务内再读):', conn2.execute('SELECT balance FROM acct WHERE id = 1').fetchone())  # (100,)
conn2.execute('COMMIT')

conn3 = sqlite3.connect(db, isolation_level=None)
print('I(新事务):', conn3.execute('SELECT balance FROM acct WHERE id = 1').fetchone())  # (50,)

# ---------- 验证 D：连接全部关闭（含 checkpoint）后重开，已提交效果仍在 ----------
conn.close(); conn2.close(); conn3.close()
c = sqlite3.connect(db)
print('D(重开库):', c.execute('SELECT * FROM acct ORDER BY id').fetchall())  # [(1, 50), (2, 100)]
c.close()
```

### 4. 常见误区与进阶思考
误区一：把 ACID 当成四条并列、由数据库独立实现的机制，认为 C 也由数据库全权保证。实际上 A/I/D 是机制，C 是目的性质：数据库只能保证被显式声明的不变量（PK / FK / UNIQUE / CHECK / 触发器 / 分区约束），而业务不变量（账户总额守恒、库存不为负、状态机合法转移）在模式定义里根本不可见。反例：若表中没有 CHECK(balance >= 0)，两个并发事务在各自视角下都合法，A/I/D 全部满足，余额却可能为负。只有当所有业务不变量都被编码为约束时，「满足约束」才与「业务一致」等价。更进一步的限制是：C 的语义是「事务前后两个状态各自满足约束」，它无法表达「执行期间不变量允许暂时被破坏」，因此很多强不变量必须靠唯一索引 + 捕获冲突，而不是先 SELECT 再 INSERT 的两步判断（后者在 RC 下必然存在竞态窗口）。

误区二：认为隔离级别越高越安全，REPEATABLE READ 等价于可串行化。REPEATABLE READ 只固定读到的版本集合，不约束写-写与读-写之间的冲突图：InnoDB 的 RR 用 MVCC 快照消除了脏读与不可重复读，用 next-key lock 消除了快照读下的幻读，但快照读不加锁，因此仍可能出现丢失更新与写偏斜；PostgreSQL 的 REPEATABLE READ 是纯 Snapshot Isolation，写偏斜必现。只有 SERIALIZABLE（SSI 冲突图检测或纯 2PL）才禁止全部串行化异常。反向的误区同样致命：隔离级别不是越高越好。RR 的间隙锁显著抬高死锁概率与锁等待时间；长事务会让 undo 版本链无法被 purge，history list length 膨胀，后续快照读需要遍历更长的版本链。工程上的正确姿势是：默认 RC，需要一致性快照的只读分析用 RR，强不变量用唯一索引 + 显式 SELECT ... FOR UPDATE 或 SERIALIZABLE 兜底，冲突路径统一走乐观重试。

思考题：undo log 本身是否受 redo 保护，为什么？把 innodb_flush_log_at_trx_commit 设为 2（提交时不 fsync redo，仅每秒批量刷盘），D 被削弱到什么程度、A 是否仍然成立？请从「原子性的物理前提是崩溃后仍能读到完整可用的 undo 前像」这一前提出发推导两件事：(1) undo 页的修改为什么必须写 redo，否则崩溃瞬间会出现什么不可恢复的状态；(2) 为什么 redo 日志必须是顺序追加 + 单条记录可校验（LSN 单调 + checksum），而不能像数据页那样原地覆盖写。
