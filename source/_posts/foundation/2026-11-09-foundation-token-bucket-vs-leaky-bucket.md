---
title: "每日基础技术总结 · 2026-11-09 · 微服务API网关限流算法：令牌桶与漏桶的区别与应用"
date: 2026-11-09 07:03:21
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-09 · 微服务API网关限流算法：令牌桶与漏桶的区别与应用

## 📚 今日主题

> **微服务API网关限流算法：令牌桶与漏桶的区别与应用**（后端基础）

### 1. 核心概念速览
**定义与本质**
- API 网关限流：在 L7 入口对请求速率/配额做准入控制（admission control），位于路由、鉴权之后，上游连接池/业务处理之前。目标不是“排队所有请求”，而是控制进入下游的速率与突发，保护 SLA、隔离租户、抑制故障放大。
- 令牌桶（Token Bucket）：维护容量为 B 的令牌桶，以速率 r 持续补充令牌；请求按成本 c 消耗令牌，不足则拒绝（policer）或延迟（shaper）。长期平均速率 ≤ r，空闲后可积累至 B，因此允许最多 B 的瞬时突发。状态为 `(tokens, last_refill_time)`。
- 漏桶（Leaky Bucket）：请求进入容量为 Q 的队列，以恒定速率 r 漏出/转发；队列满则丢弃。输出被整形为平滑速率，输入突发被队列吸收并转化为排队延迟。另一种实现是漏桶计量器（meter）：水位按 r 泄漏，请求使水位增加，超过容量拒绝；其与令牌桶存在对偶关系，工程差异在“是否排队整形”。
- 在体系中的位置：网关 ingress 的流量治理层，与鉴权、路由、熔断、并发隔离、负载削减组合；分布式限流依赖 Redis/Lua 或配额分片保证跨实例一致性。
- 为什么必须掌握：前端 `throttle/debounce` 只作用于单客户端事件循环与本地闭包状态；网关限流是跨进程共享状态的服务端准入控制，直接决定尾延迟、错误率、租户公平性与下游容量模型。

### 2. 底层原理剖析
**令牌桶底层机制**
状态：`tokens`、`last`。请求到达时刻 t、成本 c：
1. `tokens = min(B, tokens + (t - last) * r)`；
2. `last = t`；
3. 若 `tokens >= c`，则 `tokens -= c` 并放行；否则拒绝/延迟。
这是惰性补充（lazy refill），不需要后台定时器；空闲时间越长，可积累的突发额度越大，但不超过 B。长期看放行速率收敛到 r，瞬时看最多放行 B。

**漏桶底层机制**
队列版：维护 FIFO 队列与固定转发间隔 `1/r`；请求到达时若队列长度 < Q 则入队，否则丢弃；转发器每 `1/r` 取一个请求。输入突发最多被 Q 吸收，输出严格平滑；排队延迟上限约 `Q/r`，尾延迟随队列长度上升。
计量器版：维护水位 `level`，按 `level = max(0, level - (t-last)*r)` 泄漏；若 `level + c <= Q` 则接纳并 `level += c`，否则拒绝。该形式可作为 policer，容量内允许突发，但与队列版不同：它不主动延迟请求。

**与前端已有概念的对比**
- 前端 `throttle`：单线程、单客户端、定时器/闭包状态，目标是降低本页事件处理频率；网关限流：多实例、共享状态、原子读改写，目标是控制进入下游的总流量。
- 前端 `fetch` 并发池：客户端背压，恶意客户端可绕过；网关限流：服务端强制准入，必须处理 429、Retry-After、租户维度与分布式一致性。
- 前端 `debounce`：合并事件；限流算法：不合并请求，只决定接纳/拒绝/延迟。

**分布式要点**
单机令牌桶部署 N 个实例，若每实例 rate=r，则全局上限近似 N*r，不是 r。集中式实现需用 Redis Lua 将“读状态—按时间补充—扣减—写回”做成原子操作；时间源应统一或使用 Redis `TIME`，避免墙钟回拨；key 需按租户/路由维度设计并设置 TTL。高吞吐场景可用本地预取配额 + 中央配额回收，但必须处理实例故障导致的配额泄漏与流量倾斜。

### 3. 基础代码与实战验证
```text
import time
from collections import deque

class FakeClock:
    def __init__(self): self.t = 0.0
    def __call__(self): return self.t
    def advance(self, dt): self.t += dt

class TokenBucket:
    def __init__(self, rate, capacity, now=time.monotonic):
        self.rate = rate              # 每秒补充令牌数，决定长期平均速率
        self.capacity = capacity      # 桶容量，决定最大瞬时突发
        self.tokens = float(capacity) # 初始满桶：允许启动突发
        self.last = now()
        self.now = now

    def allow(self, cost=1.0):
        t = self.now()
        elapsed = max(0.0, t - self.last)   # 单调时钟差值，防止墙钟回拨
        self.tokens = min(self.capacity, self.tokens + elapsed * self.rate)
        self.last = t                       # 懒更新：无请求时不做定时补充
        if self.tokens >= cost:
            self.tokens -= cost             # 单实例内由锁/单线程保证原子扣减
            return True
        return False

class LeakyBucketQueue:
    def __init__(self, rate, capacity, now=time.monotonic):
        self.interval = 1.0 / rate          # 固定漏出间隔：输出被整形为 1/rate
        self.capacity = capacity            # 队列容量，决定可吸收的突发与排队延迟上限
        self.queue = deque()                # 存放每个已接纳请求的计划转发时刻
        self.next_departure = now()
        self.now = now

    def accept(self, now=None):
        t = self.now() if now is None else now
        if len(self.queue) >= self.capacity:
            return False                    # 队列满：直接丢弃/429，不无限排队
        depart = max(self.next_departure, t) # 第一个可立即走，后续按 interval 排队
        self.queue.append(depart)
        self.next_departure = depart + self.interval
        return True

    def pop_ready(self, now=None):
        t = self.now() if now is None else now
        ready = []
        while self.queue and self.queue[0] <= t:
            ready.append(self.queue.popleft()) # 到点才放行：输出速率恒定
        return ready

# 验证：令牌桶允许容量内突发，之后按 rate 恢复
clock = FakeClock()
tb = TokenBucket(rate=2, capacity=3, now=clock)
print([tb.allow() for _ in range(5)])  # [True, True, True, False, False]
clock.advance(0.5)                     # 0.5s * 2/s = 1 个令牌
print(tb.allow())                      # True

# 验证：漏桶队列把突发排成固定间隔，队列满则拒绝
clock2 = FakeClock()
lb = LeakyBucketQueue(rate=2, capacity=2, now=clock2)
print(lb.accept())                     # True，计划 0.0 转发
print(lb.accept())                     # True，计划 0.5 转发
print(lb.accept())                     # False，队列容量已满
print(lb.pop_ready(0.0))               # [0.0]
print(lb.pop_ready(0.4))               # []
print(lb.pop_ready(0.6))               # [0.5]
```

### 4. 常见误区与进阶思考
**误区 1：把“令牌桶允许突发、漏桶不允许突发”绝对化。** 漏桶队列版会把输入突发吸收进队列，但输出被平滑；漏桶计量器版在容量内同样允许突发。真正差异是：令牌桶以信用额度决定是否立即放行，漏桶队列版以固定转发间隔整形并引入排队延迟；计量器版与令牌桶在数学上对偶。工程选型应看下游是否能承受突发、是否允许排队延迟：突发可容忍的开放 API 常用令牌桶；对脆弱下游做恒定速率整形才用漏桶队列，且要限制 Q 防止 P99 恶化。

**误区 2：用单机限流冒充全局限流，或把限流等同于并发控制。** N 个实例各自维护令牌桶会得到 N 倍总速率；本地缓存 + 非原子读改写会产生竞态。限流控制的是速率（QPS/RPS），不能限制慢请求占满连接池/线程池；必须配合并发信号量、连接池隔离、熔断与自适应负载削减。

**思考题：** 网关有 N 个实例，采用“本地令牌桶 + 每 T 秒向 Redis 预取配额”的方案。若某实例预取后宕机，未消耗配额如何回收？若负载倾斜到少数实例，如何避免它们耗尽本地配额后大量 429，同时保证全局不超发？请给出状态机、配额回收与时钟/故障处理策略。
