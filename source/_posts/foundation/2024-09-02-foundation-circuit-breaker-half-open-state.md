---
title: "每日基础技术总结 · 2024-09-02 · 服务降级与熔断（熔断器的半开状态）"
date: 2024-09-02 20:00:00
categories: [技术分享]
tags: ["技术分享", "分布式与架构设计"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-09-02 · 服务降级与熔断（熔断器的半开状态）

## 📚 今日主题

> **服务降级与熔断（熔断器的半开状态）**（分布式与架构设计）

### 1. 核心概念速览
服务熔断（Circuit Breaker）是分布式系统中的防御性编程机制，旨在防止级联故障（Cascading Failure）。其核心本质是通过状态机管理远程依赖调用的生命周期：闭合（正常）、打开（阻断重试以保护后端）、半开（试探性恢复）。半开状态（Half-Open）是容错恢复的关键阶段，当打开定时器超时后，系统允许极少量请求通过以探测后端服务是否恢复。若探测成功，则转为闭合；若失败，则重新进入打开状态。此机制解决了在高并发场景下，因单个微服务不可用导致整个调用链雪崩的问题，确保系统的可用性和资源隔离。

在计算机体系中的位置：属于分布式一致性协议之外的可用性保障层（Availability over Consistency），常配合限流（Rate Limiting）和舱壁隔离（Bulkhead Pattern）使用。专业工程师必须掌握它，因为现代云原生架构中，网络延迟、节点宕机和服务过载是常态，缺乏熔断机制的系统在生产环境中极其脆弱。

### 2. 底层原理剖析
熔断器是一个有状态的状态机，主要包含三个状态及转换逻辑：

1. Closed (闭合): 初始状态或恢复正常后的状态。所有请求正常放行，同时统计失败率或慢调用比例。当失败阈值超过设定值时，触发熔断，状态变为 Open。
2. Open (打开): 熔断状态。所有请求直接拒绝（通常抛出异常或返回默认值），不进行底层 I/O 操作。此时系统进入冷却期（Sleep Window），等待固定时长。
3. Half-Open (半开): 冷却期结束后，状态转为 Half-Open。此时只允许极少数（如 1 个）探针请求通过下游服务。
   - 若探针请求成功：认为服务恢复，状态切换回 Closed，并重置计数器。
   - 若探针请求失败：认为服务未恢复，状态再次切换回 Open，并重置冷却计时器。

与前端/TypeScript 接口概念的对比：
前端 TS 中的 Interface 定义的是静态的契约结构（Structural Subtyping），编译时检查；而熔断器的 State Machine 定义的是动态的运行时空行为（Temporal Behavior）。TS Interface 保证代码类型安全，熔断器保证运行时韧性。TS 无法表达‘当前时刻是否允许执行’这一运行时上下文约束，这是语言层面的静态分析局限，需要运行时的策略模式来实现。

伪代码逻辑：
class CircuitBreaker {
    state = CLOSED;
    failureCount = 0;
    lastFailureTime = 0;

    method call() {
        if (state == OPEN) {
            if (now - lastFailureTime > threshold) { // 冷却时间到
                state = HALF_OPEN;
            } else {
                throw ServiceUnavailable();
            }
        }

        if (state == HALF_OPEN) {
            try {
                result = executeRemoteCall();
                state = CLOSED; // 探测成功，恢复
                resetStats();
                return result;
            } catch (e) {
                state = OPEN; // 探测失败，继续熔断
                lastFailureTime = now();
                throw e;
            }
        }

        // state == CLOSED
        try {
            result = executeRemoteCall();
            recordSuccess();
            return result;
        } catch (e) {
            recordFailure();
            if (failureRate > threshold) {
                state = OPEN;
                lastFailureTime = now();
            }
            throw e;
        }
    }
}

### 3. 基础代码与实战验证
```text
// 极简实现：验证半开状态的探测机制
// 不依赖 Spring/CircleBreaker 等框架，仅展示核心状态机逻辑

class SimpleCircuitBreaker {
    constructor(failureThreshold = 5, recoveryTimeoutMs = 5000) {
        this.failureThreshold = failureThreshold;
        this.recoveryTimeoutMs = recoveryTimeoutMs;
        this.state = 'CLOSED'; // CLOSED | OPEN | HALF_OPEN
        this.failureCount = 0;
        this.lastFailTime = 0;
        this.successCountForRecovery = 0;
    }

    // 模拟一次远程调用
    async call(targetServiceFn) {
        if (this.state === 'OPEN') {
            // 判断是否经过冷却时间
            const timePassed = Date.now() - this.lastFailTime;
            if (timePassed >= this.recoveryTimeoutMs) {
                this.state = 'HALF_OPEN';
                console.log('State transition: OPEN -> HALF_OPEN');
            } else {
                console.warn('Circuit is OPEN. Request rejected.');
                throw new Error('Service Unavailable (Circuit Open)');
            }
        }

        try {
            // 尝试执行实际逻辑
            const result = await targetServiceFn();
            
            if (this.state === 'HALF_OPEN') {
                // 在半开状态下，只要有一次成功，即视为服务完全恢复
                this.state = 'CLOSED';
                this.failureCount = 0;
                console.log('Probe succeeded in HALF_OPEN. Transition to CLOSED.');
            }
            
            return result;
        } catch (error) {
            this.failureCount++;
            this.lastFailTime = Date.now();

            if (this.state === 'HALF_OPEN') {
                // 半开状态下失败，立即重新熔断，避免无效流量冲击
                this.state = 'OPEN';
                console.log('Probe failed in HALF_OPEN. Transition back to OPEN.');
            } else if (this.failureCount >= this.failureThreshold && this.state === 'CLOSED') {
                // 闭合状态下连续失败达到阈值，开启熔断
                this.state = 'OPEN';
                console.log('Threshold reached in CLOSED. Transition to OPEN.');
            }
            
            throw error;
        }
    }
}

// 测试用例：验证半开态度的‘试错’行为
async function testHalfOpenBehavior() {
    const breaker = new SimpleCircuitBreaker(2, 1000); // 2次失败即熔断，1秒后恢复
    let callCount = 0;

    // 模拟一个最终会恢复的服务
    const mockService = () => {
        callCount++;
        // 前2次调用失败，第3次成功
        if (callCount <= 2) throw new Error('Network Timeout');
        return 'Data Received';
    };

    try {
        // 第一次：CLOSED，失败，count=1
        await breaker.call(mockService);
    } catch {}

    try {
        // 第二次：CLOSED，失败，count=2 -> 触发 OPEN
        await breaker.call(mockService);
    } catch {}

    console.log(`Current State: ${breaker.state}`); // Expected: OPEN
    
    // 等待冷却时间结束
    await new Promise(r => setTimeout(r, 1100));

    try {
        // 第三次：OPEN -> 超时 -> 变为 HALF_OPEN -> 执行 -> 成功 -> 变为 CLOSED
        const res = await breaker.call(mockService);
        console.log(`Result: ${res}, Final State: ${breaker.state}`);
    } catch (e) {
        console.error('Failed even in half-open', e);
    }
}
```

### 4. 常见误区与进阶思考
1. 误区：认为半开状态可以并发多个探针请求。
   真相：半开状态通常只允许极少量的探针（理想情况下为1个）通过，以避免在系统未完全稳定时瞬间打垮刚恢复的后端服务。如果允许多个并发探针，可能会掩盖真实的负载能力，导致再次快速熔断。

2. 误区：将熔断参数配置为静态常数，忽视业务潮汐特性。
   真相：不同接口的 SLA 差异巨大。对于写操作和非核心读操作，失败阈值和冷却时间应截然不同。静态统一配置会导致核心链路因非核心依赖抖动而频繁熔断，影响用户体验。

思考题：
在一个涉及三级依赖调用链的微服务架构中（Gateway -> Order Service -> Inventory Service -> DB），如果在 Inventory Service 对 DB 的数据库连接池耗尽导致熔断，Order Service 应当采取什么样的降级策略？是直接抛错给 Gateway，还是读取本地缓存库存？请结合‘熔断边界’与‘数据一致性’谈谈你的设计权衡。
