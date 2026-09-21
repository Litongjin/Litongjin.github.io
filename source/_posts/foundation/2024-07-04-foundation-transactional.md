---
title: "每日基础技术总结 · 2024-07-04 · @Transactional 失效的七种典型场景"
date: 2024-07-04 20:00:00
categories: [技术分享]
tags: ["技术分享", "Java 后端与 Spring 生态"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-07-04 · @Transactional 失效的七种典型场景

## 📚 今日主题

> **@Transactional 失效的七种典型场景**（Java 后端与 Spring 生态）

### 1. 核心概念速览
事务管理是数据库持久层确保数据一致性的核心机制，基于 ACID 特性（原子性、一致性、隔离性、持久性）。在 Spring 框架中，@Transactional 是通过 AOP（面向切面编程）动态代理实现的声明式事务管理。其本质是在方法执行前开启事务上下文（获取 Connection），执行后根据是否抛出受查异常（Checked Exception）或运行时异常（RuntimeException）来决定提交或回滚。

作为前端工程师出身的全栈开发者，必须理解它与前端状态管理的本质区别：前端状态通常依赖浏览器内存与 HTTP 请求的单向流动，而后端事务依赖数据库锁机制与网络往返的原子操作。Spring 的事务失效意味着 AOP 代理链断裂，导致代码虽然被调用，但未进入由 Proxy 包装的事务增强逻辑，从而失去 ACID 保障。

### 2. 底层原理剖析
Spring 的 @Transactional 依赖于 CGLIB 或 JDK Dynamic Proxy 创建的代理对象。当 Bean 被注入时，实际运行的是代理对象。代理对象拦截目标方法，检查是否存在事务属性。

失效的根本原因通常是：调用者获取的不是代理对象，而是原始对象（Target Object），因此绕过了 AOP 拦截器链。

七种典型场景的底层逻辑如下：
1. 方法非 public：JDK Proxy 仅代理 public 方法；CGLIB 虽能代理 non-public，但默认配置下可能跳过或安全策略限制，且不符合通用规范。
2. 自调用（Self-Invocation）：类内部 this.methodA() 调用 this.methodB()。由于 this 引用指向当前实例而非代理对象，methodB 上的 @Transactional 注解未被代理逻辑感知。
3. 异常类型不匹配：@Transactional 默认仅在 RuntimeException 和 Error 发生时回滚。若捕获并吞掉了 Checked Exception（如 SQLException 未包装），事务管理器视为成功，执行 Commit。
4. 异常被 catch 处理：若业务代码中 try-catch 了异常且未重新抛出（throw），事务通知中的 AfterAdvice 无法感知错误，默认执行 Commit。
5. 数据源不一致：在同一事务中切换不同的 DataSource（例如先操作主库再操作从库，或未配置多数据源事务协调器 JTA），因为每个 DataSource 对应独立的 Connection 和 TransactionManager。
6. 并发冲突超时/锁等待：高并发下若发生 Deadlock 或 Lock Wait Timeout，底层驱动会抛出特定异常。若未正确配置 propagation 行为或异常类型，可能导致事务中断但未按预期回滚其他资源。
7. 多线程调用：Spring 事务绑定在当前线程的 ThreadLocal（TransactionSynchronizationManager）中。子线程无法直接继承父线程的事务上下文，除非使用 Propagation.REQUIRES_NEW 显式创建新事务或使用异步工具类传递上下文。

### 3. 基础代码与实战验证
```text
// 演示自调用失效（Self-Invocation Failure）的底层原理
@Component
public class OrderService {

    // 这是一个普通的 Bean 实例，不是代理对象
    private OrderService self; 

    @Autowired
    public void setSelf(OrderService orderService) {
        this.self = orderService;
    }

    public void executeOrder() {
        // 【关键点1】通过 this 调用，直接访问 Target Method，绕过 Proxy
        updateInventory(); 
    }

    @Transactional(rollbackFor = Exception.class)
    public void updateInventory() {
        // 此处虽有注解，但因是自调用，Spring AOP 未介入
        // 即使抛出 RuntimeException，也不会触发 Rollback
        throw new RuntimeException("Rollback Expected but Failed");
    }
}

// 对比：修复后的标准写法，通过注入的代理对象调用
// @Autowired private OrderService orderServiceProxy;
// orderServiceProxy.updateInventory(); // 此时触发 AOP Proxy，生效
```

### 4. 常见误区与进阶思考
误区一：认为 @Transactional 可以应用于任何类或方法。事实上，它仅对同一容器内、通过接口或代理调用的 public 方法有效。私有方法 protected 方法即使有注解也无效，因为反射层面无法轻易代理且语义不明。

误区二：认为 try-catch 包裹异常不影响事务。实际上，如果在 catch 块中没有手动设置 Status.setRollbackOnly() 或再次抛出异常，Spring 的事务完成回调（CommitProcessor）会正常判断为 Success 并 Commit。

深度思考题：在使用 MySQL InnoDB 引擎时，假设一个 @Transactional 方法内执行了多条 UPDATE 语句，中间发生了 InterruptedException 导致线程终止，但 JVM 进程尚未退出。请问数据库层面的事务状态是什么？如果随后启动一个新的客户端连接查询刚才更新的数据，是否能看到部分更新的结果？为什么这涉及到分布式事务中 Saga 模式的基础考量？
