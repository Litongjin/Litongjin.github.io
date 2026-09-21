---
title: "每日基础技术总结 · 2025-02-14 · 熔断器模式的状态机：关闭、打开、半开与滑动窗口计数"
date: 2025-02-14 20:00:00
categories: [技术分享]
tags: ["技术分享", "架构与设计"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2025-02-14 · 熔断器模式的状态机：关闭、打开、半开与滑动窗口计数

## 📚 今日主题

> **熔断器模式的状态机：关闭、打开、半开与滑动窗口计数**（架构与设计）

### 1. 核心概念速览
熔断器（Circuit Breaker）是一种基于有限状态机（FSM）的容错机制，旨在防止级联故障。其核心通过监测依赖服务调用的失败率或异常频率，动态切换三种状态：1. 关闭（Closed）：正常流量经过，持续统计失败指标；2. 打开（Open）：触发阈值后拒绝所有请求，立即返回错误或降级响应，切断传播路径；3. 半开（Half-Open）：等待固定冷却时间后允许少量探针请求通过，以验证服务恢复情况。结合滑动窗口算法，实现高精度的近期行为聚合分析。该模式位于分布式系统可靠性工程的核心层，是构建高可用微服务架构的基石，也是理解异步编程、背压（Backpressure）及弹性计算的关键前置知识。

前端对比：类似前端的事件循环中的错误边界（Error Boundary）或拦截器（Interceptor），但熔断器侧重于服务端资源的保护与流量整形，而非视图渲染的稳定性。

### 2. 底层原理剖析
状态机转移逻辑遵循确定性规则，底层依赖时间片切片数据聚合：
1. 初始化：状态 Closed，重置滑动窗口计数器。
2. 运行时（Closed）：每次远程调用结束后，将结果写入当前时间片的滑动窗口。若失败计数 / 总请求数 > 阈值 Threshold，且时间间隔符合最小采样要求，则跳转至 Open。
3. 阻断期（Open）：忽略所有入站请求，记录最后切换时间 t_open。此时不执行业务逻辑，仅进行计时器比对。
4. 探测期（Half-Open）：当 currentTime - t_open >= SleepWindow 时，状态转为 Half-Open。此时限制并发数为 N（如 1 或 5），仅放行少量请求。
   - 若探针请求成功：认为服务恢复，转为 Closed，重置窗口。
   - 若探针请求失败：认为服务未恢复，转回 Open，延长休眠时间（指数退避可选）。

代码逻辑结构：
Function Execute() {
  If State == OPEN:
    Return RetryableException();
  Else If State == HALF_OPEN:
    If ProbeLimitReached: Return RetryableException();
    RecordProbeStart();

  Try:
    Result = Dependency.Call();
    OnSuccess(); // 更新窗口，标记成功
  Catch (e):
    OnFailure(e); // 更新窗口，检查阈值
  }

Function OnSuccess():
  Window.RecordSuccess();
  If State == HALF_OPEN: TransitionTo(CLOSED);

Function OnFailure(e):
  Window.RecordFailure();
  If State == CLOSED AND Window.IsFailureRateHigh():
    TransitionTo(OPEN); SetStartTime(now());
  If State == HALF_OPEN:
    TransitionTo(OPEN); ExtendSleepTime();

Function UpdateStateFromTimer():
  If State == OPEN AND now() - StartTime >= SleepWindow:
    TransitionTo(HALF_OPEN);

滑动窗口本质：使用环形缓冲区（Ring Buffer）存储最近 T 秒内的请求元组（timestamp, success），避免全量历史数据的内存开销与计算复杂度 O(N) -> O(1)。

### 3. 基础代码与实战验证
```text
/**
 * 极简熔断器状态机实现，基于 RingBuffer 滑动窗口
 */
class CircuitBreaker {
  constructor(options) {
    this.failureThreshold = options.failureThreshold; // 失败阈值，如 0.5
    this.sleepWindow = options.sleepWindow;           // 休眠窗口毫秒数
    this.windowSize = options.windowSize;            // 滑动窗口大小（最大记录数）
    
    // 状态枚举
    this.states = { CLOSED: 0, OPEN: 1, HALF_OPEN: 2 };
    this.currentState = this.states.CLOSED;
    this.lastStateChangeTime = Date.now();
    
    // 滑动窗口：存储最近的请求结果 [true|false]
    this.window = new Array(this.windowSize).fill(null);
    this.headIndex = 0;                            // 写入指针
    this.totalInWindow = 0;                        // 窗口内有效请求总数
    this.failuresInWindow = 0;                     // 窗口内失败数
    
    this.halfOpenAllowedCalls = 0;                 // 半开状态下允许的探针数
    this.maxHalfOpenCalls = 1;                     // 探针数量上限
  }

  async execute(fn) {
    // 1. 状态检查与自动迁移
    if (this.currentState === this.states.OPEN) {
      // 检查是否已过休眠期，转入半开
      if (Date.now() - this.lastStateChangeTime >= this.sleepWindow) {
        this.transitionTo(this.states.HALF_OPEN);
      } else {
        throw new Error('Circuit is OPEN');
      }
    }

    // 2. 半开状态限流
    if (this.currentState === this.states.HALF_OPEN) {
      if (this.halfOpenAllowedCalls >= this.maxHalfOpenCalls) {
        throw new Error('Circuit is HALF_OPEN: Probe limit reached');
      }
      this.halfOpenAllowedCalls++;
    }

    // 3. 执行目标函数并捕获结果
    try {
      const result = await fn();
      this.recordSuccess();
      return result;
    } catch (err) {
      this.recordFailure(err);
      throw err;
    }
  }

  recordSuccess() {
    this.updateWindow(true);
    if (this.currentState === this.states.HALF_OPEN) {
      // 半开下首个/限定请求成功，即视为恢复
      this.transitionTo(this.states.CLOSED);
    }
  }

  recordFailure(err) {
    this.updateWindow(false);
    
    // 仅在关闭状态下检测阈值，避免打开状态误判
    if (this.currentState === this.states.CLOSED) {
      if (this.getFailureRate() >= this.failureThreshold) {
        this.transitionTo(this.states.OPEN);
      }
    } else if (this.currentState === this.states.HALF_OPEN) {
      // 半开状态下失败，立即重开并可能延长休眠（此处简化为直接重开）
      this.transitionTo(this.states.OPEN);
    }
  }

  updateWindow(isSuccess) {
    // 覆盖旧数据，模拟滑动窗口
    const oldEntry = this.window[this.headIndex];
    
    if (oldEntry !== null && oldEntry === false) {
      this.failuresInWindow--;
    }
    
    this.window[this.headIndex] = isSuccess;
    this.headIndex = (this.headIndex + 1) % this.windowSize;
    
    if (this.totalInWindow < this.windowSize) {
      this.totalInWindow++;
    }
    
    if (!isSuccess) {
      this.failuresInWindow++;
    }
  }

  getFailureRate() {
    if (this.totalInWindow === 0) return 0;
    return this.failuresInWindow / this.totalInWindow;
  }

  transitionTo(newState) {
    this.currentState = newState;
    this.lastStateChangeTime = Date.now();
    this.halfOpenAllowedCalls = 0;
    // 进入 OPEN 状态时可重置部分计数器以保持准确性
    if (newState === this.states.CLOSED) {
      this.resetWindow();
    }
  }

  resetWindow() {
    for(let i=0; i<this.windowSize; i++) this.window[i] = null;
    this.totalInWindow = 0;
    this.failuresInWindow = 0;
    this.headIndex = 0;
  }
}
```

### 4. 常见误区与进阶思考
1. 误区：认为熔断器等同于超时控制（Timeout）。超时是客户端在指定时间内未收到响应则放弃连接，侧重于网络IO层面的止损；熔断器是在应用逻辑层面对连续失败行为的聚合判断，侧重于架构解耦。若无熔断，大量超时线程堆积仍会导致OOM或CPU满载。2. 误区：滑动窗口的大小选择随意。窗口过小会导致对瞬时抖动敏感（误熔断）；窗口过大则滞后性强，无法及时感知故障。需根据业务SLA的平均响应时间和故障爆发特征动态调整。

思考题：在高并发场景下，如果使用全局共享的原子计数器（AtomicInteger）而非滑动窗口来实现“最近N次失败”判断，会产生什么具体的性能瓶颈或误判风险？为什么时间维度的衰减（Sliding Window）比单纯的数量维度（Fixed Count）更适合应对突发流量波动？
