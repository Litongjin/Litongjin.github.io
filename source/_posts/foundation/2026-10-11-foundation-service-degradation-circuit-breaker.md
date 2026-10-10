---
title: "每日基础技术总结 · 2026-10-11 · 服务降级与熔断"
date: 2026-10-11 07:15:24
categories: [技术分享]
tags: ["技术分享", "分布式与架构设计"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-11 · 服务降级与熔断

## 📚 今日主题

> **服务降级与熔断**（分布式与架构设计）

### 1. 核心概念速览
服务降级（Service Degradation）是在依赖不可用、延迟升高或系统过载时，主动关闭非核心路径、简化响应或返回兜底数据，以保护核心链路容量与响应时间。服务熔断（Circuit Breaking）是调用方维护一个基于失败反馈的有状态自动开关：在时间窗口内统计对下游的失败率或失败次数，达到阈值后直接短路后续请求并快速失败，冷却期结束后进入半开探测，成功则恢复，失败则重新打开。
两者的本质不同：降级是“牺牲局部服务质量换取全局可用性”的业务策略；熔断是“根据依赖健康状态自动阻止调用”的闭环状态机。
它解决的核心问题是分布式调用链中的级联故障、线程/连接资源被慢/坏依赖耗尽、重试放大流量造成雪崩。在计算机/AI 体系中位于稳定性治理与流量防护层，与超时、重试、限流、隔离共同构成弹性工程。专业工程师必须掌握，因为前端在 BFF、SSR/边缘聚合、API 网关等场景同样面对不稳定的下游依赖，缺乏熔断的有状态反馈控制就无法实现真正的弹性。

### 2. 底层原理剖析
熔断器底层是一个三态有限状态机：

1. Closed（闭合）
   - 正常放行请求。
   - 维护统计窗口：requestCount、failureCount。
   - 判定条件：failureRate = failureCount / requestCount，且 requestCount >= minRequests（避免低流量误判）。
   - 达到阈值后切换到 Open。

2. Open（打开）
   - 不调用下游，立即返回 fallback（快速失败）。
   - 启动冷却计时 cooldownMs。
   - 冷却结束切换到 Half-Open。

3. Half-Open（半开）
   - 仅放行有限数量（通常 1 个）探测请求。
   - 探测成功：切换到 Closed，重置统计窗口。
   - 探测失败：切换到 Open，重置冷却时间。

状态迁移可由如下布尔条件表示：
  transitionToOpen      = (state == CLOSED  && failureRate >= threshold && requestCount >= minRequests)
                          || (state == HALF_OPEN && probeFailed)
  transitionToClosed    = (state == HALF_OPEN && probeSucceeded)
  transitionToHalfOpen  = (state == OPEN && now - openedAt >= cooldownMs)

统计窗口用固定窗口简化，但生产应使用滑动窗口或环形缓冲，避免窗口边界处的突刺绕过。降级策略（fallback）可以是缓存、默认值、静态兜底、备用服务、只读/异步化，它必须与熔断状态绑定，但又独立于状态机。

与前端已有概念对比：
- 前端图片 onerror 替换占位图、Promise.catch 返回默认值、React ErrorBoundary 属于“无状态单请求降级”，只处理当次失败，不维护跨请求的失败率，也无法阻止后续请求继续打到坏依赖。
- 节流/防抖是主动限频或合并，不基于失败反馈改变状态。
- 服务熔断是有状态闭环控制，基于失败反馈驱动状态迁移 + 时间冷却 + 探测恢复。其机制更接近 TCP 拥塞控制：丢包驱动拥塞窗口状态迁移，而非简单的错误兜底。

### 3. 基础代码与实战验证
```text
class CircuitBreaker {
  constructor({ failureThreshold = 3, windowMs = 10000, cooldownMs = 5000 } = {}) {
    this.state = 'CLOSED'; // 三态：CLOSED | OPEN | HALF_OPEN
    this.failureThreshold = failureThreshold; // 固定窗口内失败次数阈值
    this.windowMs = windowMs;
    this.cooldownMs = cooldownMs;
    this.windowStart = Date.now();
    this.requestCount = 0;
    this.failureCount = 0;
    this.openedAt = 0;
  }

  async call(fn, fallback) {
    const now = Date.now();

    if (this.state === 'OPEN') {
      if (now - this.openedAt >= this.cooldownMs) {
        this.state = 'HALF_OPEN'; // 冷却结束，进入半开探测
        this.resetWindow();
      } else {
        return fallback(); // 熔断打开：直接降级，不调用下游，避免资源消耗
      }
    }

    if (this.state === 'HALF_OPEN' && this.requestCount > 0) {
      return fallback(); // 半开并发控制：仅允许一个探测请求，防止冷启动打满下游
    }

    if (now - this.windowStart >= this.windowMs) {
      this.resetWindow(); // 固定窗口滚动：过期后清零统计
    }

    this.requestCount++;

    try {
      const result = await fn();
      this.handleSuccess(now);
      return result;
    } catch (err) {
      this.handleFailure(now);
      return fallback(); // 单次失败也要降级，但状态迁移由 handleFailure 决定
    }
  }

  resetWindow() {
    this.windowStart = Date.now();
    this.requestCount = 0;
    this.failureCount = 0;
  }

  handleSuccess(now) {
    if (this.state === 'HALF_OPEN') {
      this.state = 'CLOSED'; // 探测成功：状态恢复闭合
      this.resetWindow();
    }
    // CLOSED 状态成功不重置窗口，保持窗口内失败率统计连续
  }

  handleFailure(now) {
    this.failureCount++;

    if (this.state === 'HALF_OPEN') {
      this.state = 'OPEN'; // 探测失败：重新打开，并重置冷却
      this.openedAt = now;
      return;
    }

    if (this.state === 'CLOSED' && this.failureCount >= this.failureThreshold) {
      this.state = 'OPEN'; // 闭合状态失败数达到阈值：触发熔断
      this.openedAt = now;
    }
  }
}

// 验证：下游前 3 次失败，之后恢复
(async () => {
  let i = 0;
  const downstream = async () => {
    i++;
    if (i <= 3) throw new Error('downstream unavailable');
    return 'from-downstream';
  };
  const breaker = new CircuitBreaker({ failureThreshold: 2, windowMs: 1000, cooldownMs: 100 });

  for (let n = 1; n <= 5; n++) {
    const result = await breaker.call(downstream, () => 'fallback-data');
    console.log(`request ${n}: ${result}`);
  }

  await new Promise(resolve => setTimeout(resolve, 150)); // 等待冷却结束
  const result = await breaker.call(downstream, () => 'fallback-data');
  console.log(`after cooldown: ${result}`);
})();

// 预期：前 2 次失败后第 2 次触发 OPEN；第 3-5 次直接返回 fallback-data；
// 冷却 150ms 后半开探测成功，返回 from-downstream 并恢复 CLOSED。
```

### 4. 常见误区与进阶思考
常见误区：
1. 把“服务降级”等同于“熔断”：降级是返回兜底结果的业务策略，熔断是基于失败反馈的自动短路状态机。只做降级不做熔断，每次请求仍会穿透到坏依赖并消耗线程/连接/超时时间；只做熔断不设计降级，快速失败后调用方依旧拿不到可用数据。二者应组合使用。
2. 使用绝对失败次数作为阈值，忽略最小请求数和时间窗口：低流量下少量失败就误熔断，高流量下绝对阈值可能过晚或过早。正确做法是基于滑动窗口的失败率，并设置 minRequests 防止低样本抖动。

进阶思考：
在固定窗口统计中，如果故障模式是“前 5 秒全部失败，后 5 秒全部成功”，阈值 50%，窗口边界重置会导致失败率统计被清零而不触发熔断；滑动窗口如何通过重叠桶或环形缓冲避免这种边界绕过？进一步，半开探测请求若下游只是高延迟而非不可用，每个冷却期放行的探测请求仍可能堆积，此时如何用超时、并发信号量或舱壁隔离来约束探测流量？
