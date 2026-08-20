---
title: "每日基础技术总结 · 2026-06-04 · 令牌桶与漏桶限流算法"
date: 2026-06-04 08:00:00
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-06-04 · 令牌桶与漏桶限流算法

## 📚 今日主题

> **令牌桶与漏桶限流算法**（后端基础）

### 1. 核心概念速览
令牌桶（Token Bucket）与漏桶（Leaky Bucket）是两种经典的流量整形与速率控制算法，本质上是基于队列/计数器的状态机，用于约束请求或数据包在时间轴上的到达速率与突发程度。它们解决的问题是：在资源消费速率不确定或可能突发的情况下，如何在不引入全局锁或严格时间调度的前提下，以O(1)时间复杂度决策是否允许当前请求通过。令牌桶的核心机制是：以固定速率向容量为C的桶中注入令牌，每个请求消耗一个令牌，若桶中有令牌则放行，否则拒绝或等待；它允许最大C的突发（突发能力由桶容量决定），但长期平均速率被注入速率限制。漏桶的核心机制是：将请求放入一个容量为B的队列（桶），以固定速率从队列底部漏出（处理），若桶已满则新请求被丢弃或排队等待；它强制输出速率完全恒定，突发被完全平滑。在整个计算机/AI体系中，它们是服务端限流、API网关、网络QoS、消息队列背压等场景的基石算法，与滑动窗口、计数器等共同构成限流器设计的基础。专业工程师必须掌握，因为它们是理解分布式系统稳定性设计、流量控制原语以及云原生组件（如Nginx、Redis限流实现）行为的前提，且能帮助你从第一性原理评估不同限流方案的优劣，而非盲目依赖框架。

### 2. 底层原理剖析
底层运行机制可抽象为一个状态机。令牌桶：维护两个变量——令牌数tokens，上次补充时间lastRefill。每次请求到达时，首先计算当前时间now与lastRefill的时间差delta，按注入速率rate补充令牌：tokens = min(capacity, tokens + delta * rate)，并更新lastRefill = now。然后判断tokens >= 1，是则tokens--并放行，否则拒绝（或等待）。该算法通过延迟计算（lazy refill）实现O(1)复杂度，无需定时器或后台线程。漏桶：维护一个队列（或计数器）与一个恒定处理速率。请求到达时，若队列未满则入队，否则丢弃；另有一个独立的处理循环（或基于时间戳的调度）以固定速率从队列头部取出请求进行服务。实现上可用一个计数器当前水量water，以及上次排出时间lastDrain，按时间差计算应排出水量：water = max(0, water - (now - lastDrain) * drainRate)，若water < bucketSize则water++并接受，否则拒绝。这里漏桶的“排队等待”或“丢弃”策略取决于是否允许队列，若允许则用FIFO队列，若不允许则计数器模式。两者的本质区别：令牌桶允许突发且输出平均速率受限，漏桶强制恒定输出速率。与前端概念的对比：令牌桶类似浏览器中的任务调度——虽然你可以在一个事件循环里连续执行多个微任务（突发），但宏任务的总吞吐受限于每次循环的执行时间（速率）；漏桶则类似CSS动画的帧率控制——无论状态变化多频繁，渲染引擎以固定帧率输出。更精确的类比是：令牌桶是“准入控制”+“突发预算”，漏桶是“输出节流”+“平滑缓冲”。实现上，两者都可以用单调时间戳和整数运算模拟，不依赖真实时钟，这类似于前端requestAnimationFrame基于时间戳而非计数来保证时间一致性。

### 3. 基础代码与实战验证
以下为Python伪代码，用于验证令牌桶与漏桶的底层机制。关键代码行注释解释运作原理。

```python
import time

class TokenBucket:
    def __init__(self, capacity, rate):
        self.capacity = capacity        # 桶容量，允许的最大突发数
        self.tokens = capacity          # 当前令牌数，初始为满，允许立即突发
        self.rate = rate                # 每秒注入令牌数，即平均速率
        self.last_refill = time.monotonic()  # 上次补充令牌的时间戳，用于延迟计算

    def allow(self):
        now = time.monotonic()
        # 关键：根据经过的时间补充令牌，但不超过容量，这就是lazy refill
        self.tokens = min(self.capacity, self.tokens + (now - self.last_refill) * self.rate)
        self.last_refill = now          # 更新引用时间，保证下次计算的delta正确
        if self.tokens >= 1:
            self.tokens -= 1            # 消费一个令牌，即放行一个请求
            return True
        return False                    # 令牌不足，拒绝请求

class LeakyBucket:
    def __init__(self, capacity, drain_rate):
        self.capacity = capacity        # 桶容量，可容纳的最大积压请求数
        self.water = 0                  # 当前桶中的水量（积压请求数）
        self.drain_rate = drain_rate    # 漏出速率，即恒定处理速率（每秒处理的请求数）
        self.last_drain = time.monotonic()  # 上次漏水时间戳

    def allow(self):
        now = time.monotonic()
        # 关键：根据时间差排出已处理的请求，相当于队列头部出队
        self.water = max(0, self.water - (now - self.last_drain) * self.drain_rate)
        self.last_drain = now           # 更新漏水时间
        if self.water < self.capacity:  # 桶未满，允许请求进入（入队）
            self.water += 1             # 水量增加，表示请求进入队列等待处理
            return True
        return False                    # 桶满，拒绝请求（丢弃或报错）
```

验证方式：实例化TokenBucket(capacity=5, rate=1)，先连续调用5次allow()应全为True（突发），第6次立即调用应为False；等待1秒后再调用应恢复为True。LeakyBucket(capacity=5, drain_rate=1)同样连续调用5次全True，第6次False，即使等待1秒，桶中水量会减少（但不会立即恢复一个空位？实际会：1秒后drain_rate=1，water减1，因此可以再进一个），验证其恒定输出特性：若持续以10req/s速率调用，输出速率被限制为1req/s。

### 4. 常见误区与进阶思考
误区1：认为令牌桶一定比漏桶好。实际上两者应对场景不同：令牌桶允许突发，适合处理短期流量尖峰且后端能承受一定突发的情况（如API限流）；漏桶强制平滑输出，适合保护下游薄弱系统（如数据库写入、视频编码）。选错算法会导致系统在突发时被压垮或吞吐浪费。误区2：忽略时间戳精度与时钟回拨问题。实现时使用time.monotonic()而非time.time()，否则系统时钟调整会导致限流失效或误判；分布式环境下若使用不同机器的时钟，会产生偏差。另外，延迟计算必须考虑浮点误差，高并发下可用整数运算（微秒粒度）避免精度丢失。

思考题：假设令牌桶容量C=100，注入速率r=10/s，当前桶中已有50个令牌。如果在t=0时刻突然来了100个请求，然后在接下来的10秒内以每秒10个请求匀速到达，请描述令牌数量的变化过程，并计算第15秒时令牌数。这需要你精确模拟lazy refill与消费的交替，考察你是否真正理解时间驱动的补充与请求驱动的消费之间的独立关系。
