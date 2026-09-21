---
title: "每日基础技术总结 · 2024-10-18 · 接口幂等设计：Token/去重表/状态机"
date: 2024-10-18 20:00:00
categories: [技术分享]
tags: ["技术分享", "分布式与架构设计"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-10-18 · 接口幂等设计：Token/去重表/状态机

## 📚 今日主题

> **接口幂等设计：Token/去重表/状态机**（分布式与架构设计）

### 1. 核心概念速览
接口幂等性（Idempotency）指同一操作执行一次或多次对系统状态产生的影响相同。在分布式高并发场景下，它解决网络重试、消息堆积或前端抖动导致的重复请求问题。核心机制分为三类：1. Token机制：基于一次性凭证的预校验与原子删除；2. 去重表：利用数据库唯一索引构建物理层面的互斥屏障；3. 状态机：基于资源生命周期阶段约束，仅允许合法状态迁移。掌握该能力是后端工程师处理数据一致性、防止业务资损（如重复扣款、重复发奖）的基石，也是构建可靠微服务架构的核心素养。

### 2. 底层原理剖析
1. Token机制本质：将‘执行权’与‘请求参数’分离。客户端先申请Token（写入Redis或DB），携带Token发起业务请求，服务端通过原子操作（如Lua脚本或DB Delete+Where ID=Token）验证并消费Token。若Token不存在，视为非法或重复。
2. 去重表本质：利用关系型数据库的唯一约束（Unique Constraint）。建立包含 requestId/bizId/version 的唯一索引列。插入操作失败即代表冲突，无需额外锁竞争，直接返回既定结果。
3. 状态机本质：利用有限状态自动机的确定性。每个动作触发状态变更，且从状态A到B的转换路径唯一。例如订单从[待支付]到[已支付]只能执行一次，后续相同动作尝试将[已支付]转[已支付]会被状态检查拦截。
对比前端TS Interface：前端Interface定义的是静态类型契约，用于编译期检查数据结构合法性；后端接口幂等定义的是动态运行时行为契约，确保多次调用的副作用一致。前者关注结构，后者关注时序与状态。

### 3. 基础代码与实战验证
```text
// Java伪代码演示：基于Redis Lua脚本实现Token模式（保证读取与删除的原子性）
public boolean validateAndConsumeToken(String tokenKey) {
    // 定义Lua脚本：判断key是否存在且值为expectedValue，若存在则删除并返回1，否则返回0
    String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
    
    // SCRIPT_LOAD 获取SHA1值以节省带宽，EXEC 执行原子操作
    // SETNX + DEL 非原子，存在竞态条件；Lua保证单线程顺序执行无并发问题
    Long result = redisTemplate.execute(
        new DefaultRedisScript<>(script, Long.class),
        Collections.singletonList(tokenKey),
        expectedTokenValue
    );
    
    // 底层逻辑：只有当key存在且值匹配时，删除操作才会发生并返回1，否则直接返回0
    // 这确保了token只能被消费一次，杜绝了重复提交的可能性
    return result != null && result == 1;
}
// 对比DB去重表实现：
// INSERT INTO idempotent_table (request_id, status) VALUES ('req_123', 'PROCESSING')
// 如果request_id有UNIQUE KEY约束，重复插入抛出 DuplicateKeyException，捕获异常即知为重放攻击。
```

### 4. 常见误区与进阶思考
1. 误区：认为‘加了分布式锁就能实现幂等’。分布式锁（如Redisson/Redis Lock）解决的是‘并发控制’，而非‘时间维度上的重复’。如果业务耗时极短，锁粒度难以掌控，且锁释放后重试仍会进入临界区。幂等设计的核心在于‘识别’而非‘阻塞’，应先通过唯一键标识拒绝，再决定是否执行。
2. 误区：Token生成与保存不在同一事务中，导致客户端拿到Token但服务端落库失败，随后用空Token发起请求或通过其他手段伪造请求。必须保证‘发放Token’与‘标记Token已使用’的事务一致性或强原子性。
思考题：在最终一致性架构（如RocketMQ事务消息）中，如果消费者因异常导致第一次执行失败，Broker重试推送消息，此时依赖‘数据库唯一索引’作为幂等依据，是否还存在并发风险？如果存在，该如何利用版本号（CAS机制）设计更安全的重试策略？
