---
title: "每日基础技术总结 · 2025-01-27 · 分布式事务的 Saga 补偿模式与隔离性缺陷"
date: 2025-01-27 20:00:00
categories: [技术分享]
tags: ["技术分享", "架构与设计"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2025-01-27 · 分布式事务的 Saga 补偿模式与隔离性缺陷

## 📚 今日主题

> **分布式事务的 Saga 补偿模式与隔离性缺陷**（架构与设计）

### 1. 核心概念速览
Saga 是一种用于管理长期分布式事务最终一致性的模式，通过分解为一系列局部事务（Local Transactions）及其对应的补偿操作（Compensating Transactions）来替代传统的 ACID 原子性。其核心机制是：若某步骤失败，则顺序执行前置已提交步骤的逆向补偿以回滚系统状态；若成功，则直接提交。该模式牺牲了强隔离性与一致性，换取高可用性与解耦，适用于微服务架构中跨边界、长周期的业务流。专业工程师必须掌握它以理解在分布式系统中，‘事务’从数据库层面的本地锁机制演变为应用层面的状态机与时间旅行机制，这是构建高并发后端系统的基石。

### 2. 底层原理剖析
Saga 将单体事务拆解为 Saga Step 序列 {S1, S2, ..., Sn}。每个 Step i 包含正向动作 A_i 和补偿动作 C_i。

运行机制逻辑：
1. 执行 A_1，若成功，继续 A_2... 直到 A_k。
2. 若 A_k 执行失败或超时，不抛出异常终止流程，而是启动编排器（Orchestrator）或协调者（Choreographer）。
3. 反向遍历已执行的步骤 j = k-1 down to 1，依次执行 C_j 进行状态逆转。
4. 若所有 C_j 执行成功，系统回到初始一致状态（最终一致性达成）。
5. 若任一 C_j 也失败，进入人工干预或重试机制（因为 Saga 无法保证原子性，存在中间不一致窗口）。

与前端的对比：
前端 Promise/Async-Await 中的 try-catch-finally 类似 Saga，但区别在于：前端是单线程同步上下文，资源释放是即时的、局部的内存操作；Saga 是异步、跨网络、持久化存储上的状态变更。TS 的 Interface 定义契约类型，而 Saga 的定义是行为契约（Action + Compensation），且执行时序由运行时决定而非编译时。此外，前端没有‘部分提交后回滚’的概念，因为浏览器环境是无状态的 DOM 操作缓存；而 Saga 处理的是有状态的业务数据持久化，涉及脏读、幻读等隔离级别问题。

### 3. 基础代码与实战验证
```text
// 极简 Saga 执行引擎示意 (JavaScript/Node.js)
class SagaOrchestrator {
  constructor(steps) {
    // steps: [{ execute: fn, compensate: fn }]
    this.steps = steps;
  }

  async run() {
    const executedSteps = [];
    try {
      // 1. 正向执行阶段
      for (let i = 0; i < this.steps.length; i++) {
        const step = this.steps[i];
        await step.execute(); // 假设执行可能抛异常
        executedSteps.push(step);
      }
      return true; // 全部成功
    } catch (error) {
      // 2. 异常捕获触发补偿阶段
      console.error(`Step failed with: ${error.message}`);
      
      // 3. 逆向补偿阶段
      // 关键：从最后一步向前倒序执行补偿
      for (let i = executedSteps.length - 1; i >= 0; i--) {
        try {
          const step = executedSteps[i];
          await step.compensate(); // 执行补偿逻辑
        } catch (compensationError) {
          // 注意：补偿也可能失败，此时 Saga 无法自动恢复一致
          // 通常记录日志并触发告警/人工介入
          console.warn(`Compensation failed at index ${i}`, compensationError);
          throw new Error(`Irrecoverable failure in saga chain`);
        }
      }
      return false; // 虽然补偿成功，但原始事务已中断
    }
  }
}

// 使用示例
const saga = new SagaOrchestrator([
  {
    execute: async () => { /* 扣减库存 */ },
    compensate: async () => { /* 恢复库存 */ }
  },
  {
    execute: async () => { /* 创建订单 */ },
    compensate: async () => { /* 删除订单 */ }
  }
]);
```

### 4. 常见误区与进阶思考
误区一：认为 Saga 能保证强一致性。事实上，Saga 只保证最终一致性。在执行过程中到补偿完成前，系统处于‘中间不一致’状态（例如：库存已扣但未下单，或者订单已删但退款未付），其他服务在此期间查询可能读到脏数据或不完整状态。工程师需接受这种短暂的不一致，并设计相应的读取策略（如读写分离、版本控制）。

误区二：忽略补偿操作的幂等性与副作用。正向操作通常具有幂等性要求，但补偿操作若被重复执行可能导致状态翻转错误（如扣除两次库存）。必须在补偿逻辑中严格校验当前状态，确保 C_i 的执行是幂等的。

思考题：在一个基于 Saga 的转账场景中，A 账户扣款成功（Step 1），B 账户加款失败（Step 2 异常），此时正在执行 Step 1 的补偿（退回 A 余额）。如果在补偿执行期间，网络分区导致补偿消息丢失，A 的钱包服务误以为补偿未发生而再次发起请求，或者 B 服务的重试机制触发了 Step 2 的重试（尽管它本应失败），这将如何破坏 Saga 的状态机逻辑？你会如何在代码层面通过 ‘版本戳’ 或 ‘乐观锁’ 防止此类重入攻击？
