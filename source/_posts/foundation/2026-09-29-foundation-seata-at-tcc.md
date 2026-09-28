---
title: "每日基础技术总结 · 2026-09-29 · 分布式事务：Seata AT 模式与 TCC 补偿"
date: 2026-09-29 07:20:08
categories: [技术分享]
tags: ["技术分享", "Java 后端与 Spring 生态"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-29 · 分布式事务：Seata AT 模式与 TCC 补偿

## 📚 今日主题

> **分布式事务：Seata AT 模式与 TCC 补偿**（Java 后端与 Spring 生态）

### 1. 核心概念速览
Seata AT（Automatic Transaction）模式是一种基于全局锁与undo log的自动补偿型分布式事务方案，本质是两阶段提交（2PC）的工程化变体：一阶段在业务数据库本地事务中同时记录回滚日志，二阶段由全局协调器决定异步清理或反向补偿。TCC（Try-Confirm-Cancel）模式是显式补偿协议，要求业务将每个操作拆分为资源预留、确认、取消三个幂等步骤，属于业务侵入型最终一致性方案。二者均解决跨服务/跨数据源的事务一致性问题，处于分布式系统CAP理论中AP与C妥协的技术层。专业工程师必须掌握，因为微服务拆分后本地ACID失效，分布式事务成为数据一致性的最后防线；同时面对不同场景（纯数据库操作 vs 复杂业务状态）需要正确选择AT或TCC，错误选型将导致线上事故。

### 2. 底层原理剖析
AT模式工作流程：一阶段，框架解析业务SQL，在本地数据库中执行后生成前镜像与后镜像，并将undo log和业务变更放在同一本地事务中提交；同时向Seata Server注册分支事务并申请全局锁，保证其他分支不产生脏写。二阶段，若全局提交，则异步删除undo log并释放全局锁；若全局回滚，则由协调器发起分支回滚，各分支根据undo log前镜像生成反向SQL（如更新语句对应的补偿UPDATE），将数据还原为原值。此过程依赖关系型数据库的ACID特性，undo log本身也必须具备原子性。TCC模式不依赖本地事务，而是依赖业务实现Try（检查并预留资源，如冻结库存）、Confirm（用预留资源执行最终业务，如扣减库存）、Cancel（释放预留资源，如解冻库存）三方法。协调器先调用所有Try，全部成功则逐个调用Confirm；若有Try失败，则调用已成功的Try的分支的Cancel。与前端概念对比：AT的undo log相当于前端状态管理中的“快照回滚”，但需处理并发互斥（全局锁）与崩溃恢复；TCC的Try/Confirm/Cancel则类似前端“乐观更新”中“先采用临时态，失败则回退到旧态”，但TCC对每个方法都要求幂等与资源语义清晰。同时区分AT与TCC的本质：前者由框架自动补偿，业务无感知；后者由业务显式控制补偿，灵活性更高。这与Java接口和TypeScript接口的差异类似——两者都定义契约，但Java接口是运行时的类型边界，TS接口是编译期的结构约束；AT的自动行为像是编译期默认实现，TCC的手工逻辑则是运行期明确指令。

### 3. 基础代码与实战验证
```text
以下是AT回滚与TCC补偿的极简模拟，帮助理解核心机制（实际Seata依赖数据库与协调器）：

// ===== AT模式：自动记录undo log并回滚 =====
class ATBranch {
  constructor() { this.undoLog = []; }
  execute(before, after) {
    // 真实AT：在本地事务中执行业务SQL，并记录前后镜像到undo_log表
    this.undoLog.push({ before, after });
  }
  rollback() {
    // 反向遍历undo log，用前镜像生成补偿SQL
    for (let i = this.undoLog.length - 1; i >= 0; i--) {
      const { before } = this.undoLog[i];
      // 实际补偿：UPDATE t SET balance = before.balance WHERE id = before.id
      console.log('恢复为', before);
    }
  }
}

// ===== TCC模式：业务显式实现try/confirm/cancel =====
class TCCBranch {
  try() {
    // 预留资源：冻结余额，但不扣减
    this.frozen = true;
  }
  confirm() {
    // 确认提交：真正扣减余额，解冻
    this.frozen = false;
  }
  cancel() {
    // 取消补偿：解冻，回到try之前状态
    this.frozen = false;
  }
}

// 协调器逻辑：try全部成功 -> confirm；任try失败 -> 对所有已try成功分支执行cancel
```

### 4. 常见误区与进阶思考
常见误区：
1. 认为AT模式零侵入、零风险。实际上AT要求参与表必须有主键，且依赖数据库的本地事务与可回滚的SQL能力；全局锁会引入额外锁竞争与死锁窗口，高并发下性能劣化明显。
2. 将TCC的Cancel误认为简单异常处理。TCC的Try/Confirm/Cancel必须全部幂等，还要处理空回滚、悬挂、幂等控制；否则会出现预留资源被重复释放或补偿动作被错误执行。
进阶思考：Seata在TCC模式中，如果一个分支的Cancel因为网络异常而执行失败，流程会进入重试。由于网络调用是至少一次语义，Cancel可能被重复调用。假设Cancel是非幂等的（例如直接扣减用户余额），会导致资金误扣。请设计一个防重策略，基于事务ID和分支ID的唯一性来保证Cancel只生效一次，并说明在Seata的日志表中如何存储与校验这个唯一键。
