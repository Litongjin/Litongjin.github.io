---
title: "每日基础技术总结 · 2024-03-04 · RocketMQ 事务消息：Half Message 与 Op Message"
date: 2024-03-04 20:00:00
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-03-04 · RocketMQ 事务消息：Half Message 与 Op Message

## 📚 今日主题

> **RocketMQ 事务消息：Half Message 与 Op Message**（后端基础）

### 1. 核心概念速览
事务消息是 RocketMQ 实现分布式最终一致性的核心机制，其本质是通过两阶段提交协议（2PC）解耦本地事务执行与消息发送。该机制解决的是‘本地数据库操作’与‘远程消息队列持久化’之间的原子性问题，确保两者要么同时成功，要么同时回滚。Half Message（半消息）指尚未对下游消费者可见的消息状态，仅存在于 Producer 端的事务协调流程中；Op Message（操作消息/事务结果消息）是 Producer 回调后发送的 Commit/Rollback 指令。在系统架构中，它是构建高可靠异步通信链路的基础组件，专业工程师必须掌握以应对微服务间数据一致性难题，避免引入强耦合的全局锁或复杂的 saga 模式。

### 2. 底层原理剖析
RocketMQ 事务消息采用‘半消息+事务回调’的双向交互模型。底层运行逻辑如下：
1. Producer 向 Broker 发送 Half Message。Broker 将其标记为 PREPARED（不可消费）并存储。
2. Broker 响应接收成功，Producer 执行本地业务事务。
3. 根据本地事务结果，Producer 向 Broker 发送 COMMIT（提交）、ROLLBACK（回滚）或询问状态。
4. 若发送 COMMIT，Broker 将 Half Message 转换为可消费状态；若 ROLLBACK，则丢弃该消息。
5. 异常容错：若 Broker 未收到回调，启动‘回查机制’（Check），主动查询 Producer 本地事务状态。

与前端 TS 接口对比：TS 接口定义编译时约束，保证类型安全；RocketMQ 事务消息定义运行时契约，保证数据一致性。前者是静态类型系统的‘声明式’约束，后者是分布式系统中的‘过程式’补偿与协调。二者均旨在消除状态不一致的风险，但作用域不同：TS 作用于单线程内存空间，事务消息跨越网络边界与进程隔离。

### 3. 基础代码与实战验证
```text
// 基于 Apache RocketMQ 原生客户端的事务消息示例
// 关键在于 TransactionSendResult 的传递与 LocalTransactionExecuter 的实现

TransactionMQProducer producer = new TransactionMQProducer("tx_producer_group");
producer.start();

// 定义事务执行器：处理本地事务与回调
TransactionListener transactionListener = new TransactionListenerImpl();
producer.setTransactionListener(transactionListener);

Message msg = new Message("TopicTest", "TagA", "OrderID123", "Body".getBytes());

// 步骤1: 发送半消息 (Half Message)
// 此调用不直接返回正常 SendStatus，而是等待事务执行完成
SendResult sendResult = producer.sendMessageInTransaction(msg, null);

// 内部流转伪代码：
/*
 * sendMessageInTransaction -> 
 *   1. 调用 broker 发送 PREPARED 消息
 *   2. 阻塞直到 transactionListener.executeLocalTransaction(msg) 返回
 *   3. 根据返回枚举 (COMMIT/ROLLBACK) 发送 OP Message (Commit/Rollback)
 */

class TransactionListenerImpl implements TransactionListener {
    @Override
    public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        try {
            // 执行本地 DB 事务
            executeDBTransaction(msg); 
            return LocalTransactionState.COMMIT_MESSAGE;
        } catch (Exception e) {
            return LocalTransactionState.ROLLBACK_MESSAGE;
        }
    }

    @Override
    public LocalTransactionState checkLocalTransaction(MessageExt msg) {
        // Broker 回查：查询本地 DB 中订单状态，决定最终一致性
        return queryDBStatus(msg.getMsgId());
    }
}

producer.shutdown();
```

### 4. 常见误区与进阶思考
误区 1：认为发送 Half Message 后本地事务立即提交。实际上，Half Message 的可见性取决于后续 OP Message 的结果，本地事务应在判断结果后显式提交或回滚，而非依赖消息层隐式控制。

误区 2：混淆 Commit/Rollback 与消息内容修改。OP Message 不包含原始消息体，仅包含指令。Rollback 意味着物理删除或逻辑标记丢弃，不会更新消息内容。

思考题：当 Producer 在执行本地事务过程中宕机，且未发送任何 OP Message（Commit/Rollback）时，Broker 的定时任务如何保证系统不处于无限等待状态？请结合‘回查频率’与‘超时阈值’分析 Broker 的行为策略及潜在的数据一致性风险。
