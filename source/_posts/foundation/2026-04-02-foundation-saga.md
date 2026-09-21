---
title: "每日基础技术总结 · 2026-04-02 · Saga 分布式事务的补偿机制"
date: 2026-04-02 20:00:00
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-04-02 · Saga 分布式事务的补偿机制

## 📚 今日主题

> **Saga 分布式事务的补偿机制**（后端基础）

### 1. 核心概念速览
Saga 是一种用于管理长期运行分布式业务事务的无锁定协调模式，旨在解决传统两阶段提交（2PC）在高并发、跨网络边界场景下的性能瓶颈与单点故障问题。其本质并非通过预锁资源来保证原子性，而是通过‘长事务’分解为一系列本地短事务，并依赖确定的前向操作与逆向补偿操作的因果链来实现最终一致性。在计算机体系中，它是分布式系统 CAP 理论在可用性（A）与分区容错性（P）优先时的工程权衡产物。专业工程师必须掌握它，因为现代微服务架构天然割裂了数据局部性，传统的数据库级 ACID 无法直接跨服务生效，必须从应用层构建业务语义级的回滚机制以处理网络抖动、服务宕机等不可靠因素。

principals, "core_mechanism": "Saga 的核心机制是将一个全局事务 T 拆分为子事务 T1, T2, ..., Tn，每个 Ti 对应本地服务的独立操作 Oi。关键约束在于：若所有 Oi 成功执行则全局提交；若任何 Ti 失败或超时，则触发反向补偿序列 C_{i-1}, C_{i-2}, ..., C_1，其中 Ci 是 Oi 的严格逆操作（Inverse Operation）以确保状态回退至初始点。实现流程通常由 Saga 编排器（Choreography 或 Orchestration）控制：1. 顺序执行正向操作并持久化日志；2. 捕获异常或监控心跳；3. 若需补偿，按逆序调用补偿 API。这与前端中‘撤销/重做（Undo/Redo）’栈机制类似，但区别在于 Saga 的补偿具有副作用（Side Effects），如资金扣除、库存扣减等物理世界或外部系统的变更，且要求补偿操作具备幂等性（Idempotency）和最终可达性，而前端 Undo 通常仅作用于内存对象树，不涉及 IO 延迟与分布式通信的不确定性。",

code": "// 极简伪代码演示 Orchestration 模式下的 Saga 引擎逻辑
// 注意：此处省略 RPC 调用细节，聚焦于补偿链的执行逻辑

class SagaEngine {
  // 子事务结构体，包含正向执行函数与对应的逆向补偿函数
  struct Step { action: () => Promise<Result>, compensate: () => Promise<void> }
  private steps: Step[] = []
  private executedIndices: Set<int> = new Set()

  async executeSaga(): Promise<void> {
    try {
      // 正向执行阶段：串行提交本地事务
      for (let i = 0; i < this.steps.length; i++) {
        const result = await this.steps[i].action();
        if (!result.success) throw new Error(`Step ${i} failed`);
        this.executedIndices.add(i); // 记录已完成的步数，用于精确补偿
      }
      // 所有步骤成功，隐式提交全局事务
    } catch (error) {
      // 错误发生，进入补偿阶段：逆序执行已执行步骤的补偿操作
      console.error('Transaction rolled back, initiating compensation...');
      let lastCompensatedIndex = -1;
      // 逆序遍历已执行的步骤
      for (let i = this.executedIndices.size - 1; i >= 0; i--) {
        const index = Array.from(this.executedIndices)[i];
        try {
          // 关键点：补偿操作必须处理部分失败的情况，确保最终一致性
          await this.steps[index].compensate();
          lastCompensatedIndex = index;
        } catch (compError) {
          // 如果连补偿都失败了，必须记录到死信队列或人工干预，
          // 因为此时系统处于‘中间状态’，不能简单忽略
          console.error(`Compensation for step ${index} failed manually介入 required`, compError);
        }
      }
      throw error; // 重新抛出原始错误供上层处理
    }
  }
}

// 业务定义示例
const sagaSteps = [
  new Step(
    action: () => createOrder(), // 本地 DB 插入订单
    compensate: () => cancelOrder() // 本地 DB 删除或标记删除订单
  ),
  new Step(
    action: () => deductInventory(), // 远程调库存服务扣减
    compensate: () => restoreInventory() // 远程调库存服务恢复库存
  )
];"
pitfalls": "1. **补偿非逆运算误区**：许多初学者认为补偿只是简单地把数据改回去，但在真实场景中，如‘发送邮件’，邮件一旦发出无法真正‘收回’，因此补偿操作应定义为‘发送取消通知邮件’或‘标记用户免打扰’，即业务语义上的抵消而非数据字段的镜像反转。这要求设计者深刻理解业务边界。
2. **时序与脏读风险**：在执行正向操作 A 后，若系统宕机重启，Saga 引擎可能不知道 A 是否最终成功（未提交日志）。如果在补偿逻辑中假设 A 已完成而执行 B，可能导致逻辑错误。因此，每个子事务的执行必须带有唯一的事务 ID（TraceID），且补偿接口必须强幂等，基于全局状态而非局部参数进行决策。进阶思考题：在一个包含三个服务的 Saga 链路中，若第二个服务在执行正向操作期间因网络分区导致超时而返回‘未知’状态（Non-deterministic outcome），作为架构师，你该如何设计重试策略与补偿触发阈值，以避免产生僵尸事务或重复补偿导致的资损？"}
