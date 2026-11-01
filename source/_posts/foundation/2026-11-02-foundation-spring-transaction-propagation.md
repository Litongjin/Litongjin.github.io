---
title: "每日基础技术总结 · 2026-11-02 · Spring 事务传播行为"
date: 2026-11-02 07:01:51
categories: [技术分享]
tags: ["技术分享", "Java 后端与 Spring 生态"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-02 · Spring 事务传播行为

## 📚 今日主题

> **Spring 事务传播行为**（Java 后端与 Spring 生态）

### 1. 核心概念速览
**定义**：Spring 事务传播行为（Transaction Propagation）指：当一个被 @Transactional 声明的方法在**已存在事务上下文**（当前线程的 ThreadLocal 中已绑定 ConnectionHolder）中被调用时，用于决定「被调方法如何参与或处置这个已存在的物理事务」的规则枚举。取值见 org.springframework.transaction.annotation.Propagation，共 7 种：REQUIRED、REQUIRES_NEW、NESTED、SUPPORTS、NOT_SUPPORTED、MANDATORY、NEVER。

**本质**：它不是「事务」，而是一套**事务边界的组合规则**：在单线程调用栈上，对 ThreadLocal 绑定的物理资源（JDBC Connection）与逻辑边界（commit / rollback / Savepoint）做四类决策——新建、加入、挂起、禁止。谁负责终止边界、谁只是参与者、谁需要独立原子性、谁必须与外层隔离，全部由该枚举决定。

**解决的问题**：分层服务调用（Service A → Service B → Service C）中，**物理事务数量与逻辑方法边界数量并非一一对应**。传播行为定义了边界所有权；否则嵌套调用要么丢失原子性（内层各自提交），要么过度合并（内层失败拖垮可容忍的外层流程），要么产生不可预期的隔离缺失。

**机制**：Spring AOP 代理 + TransactionInterceptor + PlatformTransactionManager.getTransaction(TransactionDefinition)；AbstractPlatformTransactionManager 依据 isExistingTransaction() 与 propagationBehavior 分支，决定是否 doBegin（setAutoCommit(false)）、是否 doSuspend/doResume、是否创建 Savepoint。

**体系位置**：Java 后端服务层与持久层的核心契约，横跨四层基础：Spring AOP（代理模型与拦截时机）、ThreadLocal（资源绑定与上下文传递）、JDBC（Connection 生命周期、Savepoint、autoCommit）、数据库（隔离级别、行锁、MVCC）。它决定事务原子性的**真实边界**，进而决定并发正确性与连接池占用形态；不理解它，@Transactional 只是「看起来原子」的注解。

### 2. 底层原理剖析
**一、执行链（全部发生在单线程调用栈上）**

调用方 → 代理对象 → TransactionInterceptor.invoke() → createTransactionIfNecessary() → PlatformTransactionManager.getTransaction(def) → 目标方法体（执行 SQL）→ commit/rollback → cleanupAfterCompletion()（清理 ThreadLocal 绑定与 Synchronization）。

关键点：传播行为的判定**只发生在进入方法体之前那一次决策**中；方法体内部的 SQL 不参与决策，方法体也无法改变已做出的决策。

**二、getTransaction 决策树（精确伪代码）**

txObject = doGetTransaction()  // DataSourceTransactionManager：TransactionSynchronizationManager.getResource(dataSource)，即 ThreadLocal 中的 ConnectionHolder

if (txObject.hasExistingTransaction()) {   // 当前线程已绑定 Connection 且事务激活
    handleExistingTransaction(def, txObject);
} else {
    if (propagation == MANDATORY) throw new IllegalTransactionStateException();
    if (propagation == NEVER) 正常以无事务方式执行；
    startTransaction(): con.setAutoCommit(false); 绑定 ConnectionHolder 到 ThreadLocal; newTransaction = true;
}

handleExistingTransaction 的 switch 即为传播行为的全部本质：

REQUIRED      → 加入：返回参与者句柄（newTransaction=false），不新建连接、不 commit/rollback；但内层抛异常会把**整个物理事务**标记 rollback-only。
SUPPORTS      → 有则加入，无则以无事务方式执行（连接保持 autoCommit=true）。
MANDATORY     → 有则加入，无则抛异常（契约校验，用于强制「必须被外层事务包裹」）。
REQUIRES_NEW  → doSuspend(外层资源) → 新建独立物理事务（新 Connection、新 autoCommit=false）→ 内层自行 commit/rollback → doResume(外层资源)。
NOT_SUPPORTED → doSuspend → 以无事务方式执行 → doResume。
NEVER         → 存在事务则抛异常。
NESTED        → 若无外层事务则退化为 REQUIRED；否则 con.setSavepoint()，内层失败 rollback(savepoint)，外层事务继续；管理器不支持保存点时抛 NestedTransactionNotSupportedException（JPA 默认如此）。

**三、挂起/恢复的真实动作**

doSuspend()：从 TransactionSynchronizationManager 解绑 ConnectionHolder 与全部 TransactionSynchronization，封装进 SuspendedResourcesHolder；doResume()：重新绑定。
要点：**挂起不等于提交，也不等于关闭**。外层 Connection 依然存活，持有其未提交写入与已加的行锁，只是当前线程暂时不可见它。所以「隔离」成立于可见性与锁的层面，而代价是同一调用栈内需要**第二条连接**（连接占用翻倍）。

**四、回滚语义的三个层级（必须区分）**

1) 物理事务回滚：Connection.rollback()，撤销全部未提交写入，释放锁，结束事务。
2) 保存点回滚：Connection.rollback(savepoint)，只撤销保存点之后的写入；事务仍在，**锁不因保存点回滚而释放**，隔离级别不变。
3) 全局标记 globalRollbackOnly：在 REQUIRED 下内层失败时，参与者没有独立物理边界可用于局部回滚，管理器只能把整个事务标记 rollback-only；外层若继续 commit，则抛 UnexpectedRollbackException。这是「逻辑边界缺少物理边界」时的必然结果，不能靠 try-catch 吞掉异常来绕过。

**五、与前端已有体系的对照**

1) 与 Node 的 AsyncLocalStorage / zone.js 同构：都是「执行上下文沿调用链传递」。差异在于 Java 侧上下文是**显式 ThreadLocal 绑定**，不随 Promise/线程池自动传播：@Async、CompletableFuture、new Thread 都会使事务上下文丢失，新线程上传播行为退化为「无外层事务」（REQUIRED 会新建物理事务）。
2) 与 JS Proxy 同构的失效模式：Spring AOP 代理是「容器注入时替换引用」，拦截器只在调用穿过代理时生效；this.b() 自调用绕过代理，等价于 const f = proxy.method; f() 丢失 trap。区别仅在实现层次：Java 走字节码层面的接口实现/子类包装，JS 走运行期对象行为拦截。
3) 与 React Context 的本质区别：Context 只解决「读取型数据传递」，不涉及资源所有权与生命周期终止；事务传播决策的是**控制权**——谁负责边界终止，而非数据怎么拿到。
4) 三个正交维度：propagation 决定**物理事务数量与边界归属**；isolation 决定**并发可见性**；锁由数据库在语句执行时施加。传播行为不改变隔离级别，也不减少锁竞争——REQUIRES_NEW 反而是锁冲突与连接耗尽的常见来源。

### 3. 基础代码与实战验证
```text
以下为不依赖 Spring 的最小可运行模型：用 ThreadLocal 绑定 Connection 模拟 TransactionSynchronizationManager，用 Savepoint 模拟 NESTED，用解绑/重绑模拟 suspend/resume。

public final class MiniTxManager {

    // 物理事务上下文：Connection 绑定在当前线程上（等价于 Spring 的 TransactionSynchronizationManager）
    private static final ThreadLocal<Connection> HOLDER = new ThreadLocal<>();
    private final DataSource dataSource;

    public MiniTxManager(DataSource ds) { this.dataSource = ds; }

    // 对应 isExistingTransaction()：判定「当前线程是否已有激活的物理事务」
    private boolean existing() { return HOLDER.get() != null; }

    // PROPAGATION_REQUIRED：有则加入（不新建、不提交、不回滚），无则新建
    public void required(TxCallback cb) throws Exception {
        if (existing()) {              // 加入：连接不变，内层无边界控制权
            cb.run(HOLDER.get());      // 内层异常直接向上抛，由外层物理事务统一 rollback
            return;                    // 关键：此返回不代表「已提交」，只代表「边界不归我管」
        }
        Connection c = dataSource.getConnection();
        c.setAutoCommit(false);        // 物理事务真正开始的唯一动作
        HOLDER.set(c);                 // 绑定到线程，后续同线程方法据此判定 hasExistingTransaction
        try {
            cb.run(c);
            c.commit();                // 仅边界所有者提交
        } catch (Exception e) {
            c.rollback();              // 撤销该连接上全部未提交写入
            throw e;
        } finally {
            HOLDER.remove();           // 必须解绑，否则线程池复用线程会串事务
            c.setAutoCommit(true);
            c.close();
        }
    }

    // PROPAGATION_REQUIRES_NEW：挂起外层上下文，用第二条连接开独立物理事务
    public void requiresNew(TxCallback cb) throws Exception {
        Connection outer = HOLDER.get();          // 取出并保存待恢复的资源
        HOLDER.remove();                          // doSuspend()：仅解绑，外层连接仍持有未提交数据与行锁
        Connection c = dataSource.getConnection(); // 新连接 = 新物理事务：不共享可见性，也不共享 Savepoint
        c.setAutoCommit(false);
        HOLDER.set(c);
        try {
            cb.run(c);
            c.commit();                           // 内层独立提交，不受外层后续回滚影响
        } catch (Exception e) {
            c.rollback();
            throw e;
        } finally {
            HOLDER.remove();
            c.setAutoCommit(true);
            c.close();
            if (outer != null) HOLDER.set(outer); // doResume()：恢复外层上下文
        }
    }

    // PROPAGATION_NESTED：同一物理事务内用 Savepoint 划出子边界
    public void nested(TxCallback cb) throws Exception {
        Connection c = HOLDER.get();
        if (c == null) { required(cb); return; }  // 无外层事务时退化为 REQUIRED
        Savepoint sp = c.setSavepoint();          // 数据库侧保存点，事务与锁均不释放
        try {
            cb.run(c);
        } catch (Exception e) {
            c.rollback(sp);                       // 只撤销 sp 之后的写入，外层此前写入保留
            throw e;                              // 是否中断外层由调用者决定（与 REQUIRED 的 rollback-only 不同）
        }
    }

    public interface TxCallback { void run(Connection c) throws Exception; }
}

验证用 main（观察写入结果即可验证三种语义差异）：

MiniTxManager tx = new MiniTxManager(dataSource);
tx.required(c -> {
    insert(c, "A");                                   // 外层物理事务内的写入，未提交
tx.nested(c2 -> {
        insert(c2, "B");
        throw new RuntimeException("inner fail");       // 触发 rollback(savepoint)
    });
    insert(c, "C");                                   // 保存点已回滚，外层继续执行
});
// 结果：A、C 提交；B 被撤销。换成 requiresNew 且内层成功提交，则 B 独立落库（外层后续回滚也不影响 B）。
// 换成 required 且内层异常向上抛，则 A、C 一并回滚——参与者没有独立物理边界。

// 隔离性验证：外层已 insert A 未提交，内层 requiresNew 查询看不到 A（READ COMMITTED）；
// 若内层 update 同一行 A，则被外层持有的行锁阻塞——挂起不等于提交，锁不随逻辑边界释放。
```

### 4. 常见误区与进阶思考
**误区一：把 NESTED 当作 REQUIRES_NEW，或以为 NESTED 普遍可用**
NESTED 只有**一个物理事务**，依赖 JDBC Savepoint：内层失败只回滚到保存点，外层仍可提交，但内层已获取的行锁、已延长的长事务、已消耗的 undo 均不会因保存点回滚而消失；且 JpaTransactionManager 默认不支持保存点，直接抛 NestedTransactionNotSupportedException。REQUIRES_NEW 则是**两条连接、两个物理事务**：可独立提交，也能让内层看不到外层未提交数据，代价是连接占用翻倍（池中连接数不足时形成等待链，甚至池大小为 1 时自锁）。把两者混用的典型后果是：期望「内层失败不影响外层」却整体回滚，或期望「内层写入随外层回滚」却已独立提交。

**误区二：以为 catch 住内层异常就能让外层照常提交**
REQUIRED 下内层是参与者，抛出的运行时异常会把物理事务标记 globalRollbackOnly；此后外层即使没有异常，提交时也会抛 UnexpectedRollbackException。这不是 Spring 的过度设计，而是本质约束：内层已执行的 SQL 与可能已部分失效的语句无法在「逻辑边界」上局部撤销，唯一安全的动作是整体回滚。要真正实现局部容错，必须用 REQUIRES_NEW（独立物理事务）或 NESTED（保存点），而不是 try-catch 掩盖异常。

**附带高频坑（并列提醒）**：同类内部 this.b() 自调用绕过代理，传播行为与 @Transactional 全部不生效，等价于把方法从 Proxy 上单独取出调用而丢失 trap；跨线程（@Async、线程池、CompletableFuture）时 ThreadLocal 不传递，传播语义在新线程上退化为「无外层事务」。

**进阶思考题**：外层 REQUIRED 事务已 UPDATE 表 T 中主键 id=1 的行但尚未提交；随后在同一线程内调用一个 REQUIRES_NEW 方法，该方法也 UPDATE 同一行 id=1。请依据挂起机制、行锁持有者与连接归属，推断实际发生什么？内层会「读到外层的未提交值」吗？会「阻塞」吗？阻塞的等待方与被等待方分别是谁？如果把它改为 NESTED，行为是否改变？——能准确说清「挂起不等于提交、锁随物理连接而非逻辑边界存在、同一线程内的两条连接会互相等待」这三点，才算真正掌握传播行为的底层机制。
