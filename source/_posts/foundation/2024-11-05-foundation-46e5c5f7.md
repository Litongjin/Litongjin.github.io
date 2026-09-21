---
title: "每日基础技术总结 · 2024-11-05 · 限流算法：令牌桶、漏桶、滑动窗口"
date: 2024-11-05 20:00:00
categories: [技术分享]
tags: ["技术分享", "分布式与架构设计"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-11-05 · 限流算法：令牌桶、漏桶、滑动窗口

## 📚 今日主题

> **限流算法：令牌桶、漏桶、滑动窗口**（分布式与架构设计）

### 1. 核心概念速览
限流算法是分布式系统中保障系统稳定性与资源可控性的核心机制，本质是对并发请求流量进行整形（Shaping）或过滤（Filtering），防止上游突发流量压垮下游服务。

1. 令牌桶（Token Bucket）：允许突发流量。系统以固定速率向桶中填充令牌，请求需获取令牌才能执行；桶满时丢弃后续令牌。特点是支持一定程度的突发（Burst），符合大多数 Web 场景需求。
2. 漏桶（Leaky Bucket）：强制平滑流量。请求进入容量固定的桶中，以固定速率流出处理。无论输入速率如何，输出速率恒定。主要用于保护后端服务免受瞬时峰值冲击，适用于对响应时间一致性要求极高的场景。
3. 滑动窗口（Sliding Window）：基于时间的精确统计。将时间轴划分为多个小窗口，记录每个窗口的请求数，通过加权计算当前时间片内的总请求量。相比固定窗口，它消除了边界效应（Boundary Effect），实现更平滑的限流效果。

在 AI 体系位置：在 LLM 推理服务中，令牌桶常用于控制 Token 生成速度（TPS/QPS），漏桶用于保护 GPU 显存不被 OOM，滑动窗口用于监控 API 调用的长期趋势。专业工程师必须掌握，因为这是从单体应用走向高可用分布式架构的基石，直接决定系统的 SLA（服务等级协议）表现。

### 2. 底层原理剖析
核心机制差异剖析：

1. 令牌桶 vs 漏桶：
   - 状态维护：令牌桶维护剩余令牌数（初始为桶容量 C，随时间增加 min(T + rate, C)）；漏桶维护当前水量（请求进入增加，处理减少 min(W - drain, 0)）。
   - 流量特性：令牌桶允许突发，只要桶内有足够令牌即可瞬间消费大量请求；漏桶强制匀速输出，无论请求多快，处理速率恒定为 drain_rate。
   - 前端对比：类似 React 的状态管理。令牌桶像 Redux，关注最终状态是否满足条件（有令牌）；漏桶像受控组件（Controlled Component），严格限制更新频率和幅度。

2. 固定窗口 vs 滑动窗口：
   - 精度问题：固定窗口将时间划分为离散区间（如每秒一个窗口），在窗口切换临界点可能产生两倍于阈值的流量（例如第 0.9s 接收 N/2 请求，1.1s 又接收 N/2 请求）。
   - 平滑处理：滑动窗口将上一个窗口的计数按重叠比例衰减并合并到当前窗口计算。公式近似为：current_count = prev_window_count * (1 - elapsed_ratio) + curr_window_count。这消除了临界点突刺，实现了连续时间的线性插值估计。

3. 底层实现难点：
   - 分布式一致性：上述算法在单机可用 ConcurrentHashMap 或 AtomicInteger 实现。但在分布式环境下，需借助 Redis Lua 脚本保证原子性，或使用 Redisson 等库。核心挑战在于时钟同步（NTP）和分布式锁的开销。
   - 内存模型：滑动窗口若采用精确日志（如 Sorted Set），内存占用随时间线性增长；若采用近似算法（如 Google Guava 的 RateLimiter），则牺牲少量精度换取极低 CPU 开销。

### 3. 基础代码与实战验证
```text
/**
 * 极简令牌桶实现原理演示（基于 Java 伪代码逻辑）
 * 注：生产环境请勿手写，应使用 Guava RateLimiter 或 Redis+Lua
 */
public class SimpleTokenBucket {
    private final int capacity; // 桶容量
    private double permits;     // 当前令牌数
    private long lastTime;      // 上次补充令牌的时间戳（毫秒）
    private final double rate;  // 令牌生成速率（每秒多少个）

    public SimpleTokenBucket(int capacity, double rate) {
        this.capacity = capacity;
        this.permits = capacity; // 初始装满
        this.rate = rate;
        this.lastTime = System.currentTimeMillis();
    }

    /**
     * 尝试获取 n 个令牌
     * @return true 如果成功获取，false 否则
     */
    public synchronized boolean tryAcquire(int n) {
        long now = System.currentTimeMillis();
        // 1. 计算时间段内新增的令牌数
        // 机制：令牌不是预先生成的，而是惰性计算（Lazy Evaluation），节省资源
        double newTokens = (now - lastTime) / 1000.0 * rate;
        
        // 2. 更新桶中令牌总数，上限不超过容量
        this.permits = Math.min(this.capacity, this.permits + newTokens);
        this.lastTime = now;

        // 3. 检查是否有足够令牌
        if (this.permits >= n) {
            // 4. 扣除令牌，返回成功
            this.permits -= n;
            return true;
        } else {
            // 5. 令牌不足，拒绝请求
            return false;
        }
    }
}

/**
 * 关键注释解释：
 * - lastTime 的作用：避免每次调用都重新生成所有历史令牌，仅计算自上次调用以来的增量。
 * - synchronized 的作用：保证多线程下 permits 和 lastTime 的原子更新，防止竞态条件导致令牌超发。
 * - 与前端的区别：前端节流（Throttle）通常基于客户端事件触发，服务端限流必须基于服务器真实处理能力，且需考虑网络延迟和服务节点间的负载不均。
```

### 4. 常见误区与进阶思考
1. 误区：认为 "QPS 限制" 等同于 "令牌桶"。实际上 QPS 限制可以是固定窗口、滑动窗口或令牌桶等多种实现。混淆概念会导致选型错误，例如在需要处理突发流量的 CDN 场景中误用漏桶，会导致用户体验下降。

2. 误区：忽略分布式时钟偏差。在集群部署中，如果各节点未通过 NTP 严格同步时间，基于本地时间的限流算法会出现严重漂移，导致整体限流策略失效或过度限流。解决方案是使用集中式时间源或 Redis 原子操作代替本地时钟。

3. 深度思考题：在设计一个大语言模型（LLM）API Gateway 时，如果我们需要同时保证 'GPU 计算资源的稳定消耗' 和 '用户感知的低延迟首字响应（TTFT）'，你会如何组合使用令牌桶、漏桶和滑动窗口？请画出数据流向图并说明理由。（提示：考虑生成阶段与解析阶段的解耦。）
