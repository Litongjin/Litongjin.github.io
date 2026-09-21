---
title: "每日基础技术总结 · 2025-09-29 · Spring 事务传播行为（REQUIRED/REQUIRES_NEW 等）"
date: 2025-09-29 20:00:00
categories: [技术分享]
tags: ["技术分享", "Java 后端与 Spring 生态"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2025-09-29 · Spring 事务传播行为（REQUIRED/REQUIRES_NEW 等）

## 📚 今日主题

> **Spring 事务传播行为（REQUIRED/REQUIRES_NEW 等）**（Java 后端与 Spring 生态）

### 1. 核心概念速览
Spring 事务传播行为（Transaction Propagation）定义了在一个事务方法被另一个事务方法调用时，事务上下文如何传递与合并的语义机制。其本质是基于 ThreadLocal 的事务状态管理在方法调用边界上的状态切换策略。它解决的核心问题是：当业务逻辑发生嵌套或并行调用时，如何保证数据的一致性与隔离性边界。在计算机体系结构中，它对应于分布式系统或模块间调用的‘原子性’契约定义；对专业工程师而言，掌握它是理解并发控制、资源竞争以及微服务链路中数据一致性问题的基石，因为错误的传播行为会导致非预期的数据提交或回滚，破坏系统的 ACID 属性。

### 2. 底层原理剖析
底层实现基于 AOP 代理链与 PlatformTransactionManager 的交互。核心数据结构是 TransactionSynchronizationManager 中的 ThreadLocal 变量，用于存储当前线程绑定的 TransactionInfo（包含事务连接、挂起状态等）。

1. REQUIRED (默认): 若当前存在事务则加入；若不存在则新建。机制上是状态的‘复用或创建’。流程：检查 ThreadLocal -> 有则 join(); 无则 begin()。
2. REQUIRES_NEW: 总是新建独立事务。机制上是‘挂起与恢复’。流程：检查 ThreadLocal -> 若有，保存旧连接并 suspend() -> 开启新事务 -> 执行完 commit/rollback -> 恢复旧事务 resume()。
3. NESTED: 在现有事务内部创建一个子事务点（Savepoint）。机制上是‘部分回滚隔离’。外层提交则子事务必须提交，否则回滚到 Savepoint；外层回滚则整体回滚。依赖于 JDBC Savepoint 或 JTA 支持。

与前端对比：前端 Promise.all (parallel) / Promise.race (compete) 处理的是异步结果的聚合失败模式，而 Spring 传播行为处理的是同步调用栈中资源锁定状态的层级继承与隔离。TS 接口只定义类型契约（静态），Spring 传播行为定义运行时行为契约（动态），前者约束编译期结构，后者约束运行期资源生命周期。

### 3. 基础代码与实战验证
```text
// 关键验证代码片段：展示 REQUIRES_NEW 的挂起与恢复机制
@Transactional(propagation = Propagation.REQUIRED)
public void outerMethod() {
    // 1. 开启事务 T1, 绑定到当前 ThreadLocal
    Connection conn1 = dataSource.getConnection(); 
    
    innerMethod(); 
    
    // 若 inner 抛出 RuntimeException，T1 标记为 rollbackOnly
    // 2. 尝试提交 T1
    try { conn1.commit(); } catch (...) { conn1.rollback(); }
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void innerMethod() {
    // 1. 检测到 ThreadLocal 已有 T1
    // 2. Suspend T1: 断开 conn1 与线程的绑定，将 conn1 暂存
    // 3. Begin T2: 获取新连接 conn2 并绑定到线程
    Connection conn2 = dataSource.getConnection();
    
    try {
        // 业务逻辑执行
    } finally {
        // 4. Commit/Rollback T2
        conn2.commit(); 
        // 5. Resume T1: 将 conn1 重新绑定回 ThreadLocal
    }
}
```

### 4. 常见误区与进阶思考
误区1：认为 REQUIRES_NEW 能拦截外层方法的异常回滚。事实上，REQUIRES_NEW 开启的新事务独立于外层，若内层抛出未捕获异常，仅内层事务回滚，外层事务若未捕获该异常继续执行，仍可能因自身逻辑错误触发自身回滚，但内层已提交的操作不会回滚。只有在外层显式 try-catch 或内层配置 rollbackFor 且外层继续运行时，才需警惕这种‘部分成功’现象。
误区2：混淆 NESTED 与 REQUIRES_NEW。NESTED 共享同一个数据库连接，依赖 Savepoint，性能开销较小但受限于 DB 特性；REQUIRES_NEW 使用独立连接，支持跨数据源，但上下文切换开销大。
深度思考题：若在 @Transactional 方法中通过编程方式直接调用本类中的另一个 @Transactional 方法（Self-Invocation），为何传播行为会失效？请从 Spring AOP 的 CGLIB/JDK Proxy 代理机制及字节码增强角度解释其底层原因。
