---
title: "每日基础技术总结 · 2026-07-10 · Jaeger 的采样策略：头部采样与尾部采样及降采样"
date: 2026-07-10 08:00:00
categories: [技术分享]
tags: ["技术分享", "DevOps 与云原生"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-10 · Jaeger 的采样策略：头部采样与尾部采样及降采样

## 📚 今日主题

> **Jaeger 的采样策略：头部采样与尾部采样及降采样**（DevOps 与云原生）

### 1. 核心概念速览
Jaeger 的采样策略是分布式追踪链路中控制数据体量与信息完整度之间权衡的机制，贯穿探针（jaeger-client）与收集端（collector/ingester）两层，属于可观测性（Observability）体系中『成本控制与数据有效性』的核心子领域。头部采样（head-based sampling）指在 trace 的根 span 创建时、即链路尚未展开之前，由客户端 tracer 依据概率、限流或远端下发的策略做出二元决策（采/不采），该决策通过 trace context（如 uber-trace-id 的 flags 位或 W3C traceparent）随请求逐跳传播，子 span 无条件继承，因此整条 trace 要么全采、要么全弃。尾部采样（tail-based sampling）指在 span 到达收集端之后，基于完整链路特征（错误状态、端到端延迟、状态码）对已缓冲的 trace 做出决策，弥补头部采样『决策早于事实』的盲区。降采样（downsampling）是对采样率的动态抑制或二次过滤，包括客户端自适应采样（collector 按 QPS 为每个 operation 计算下发采样率）与收集端的 trace 级丢弃，目的是在突发流量下维持采集成本上限。专业工程师必须掌握它，是因为采样策略直接决定延迟百分位、错误率等 SLO 指标的统计无偏性：配置错误会导致监控数据失真、存储成本失控，且问题往往在系统高负载时才暴露。

### 2. 底层原理剖析
头部采样的底层机制是一个确定性的纯函数，以 trace_id 为唯一输入。概率采样实现为：sampled = hash(trace_id) / 2^64 < param，param 为采样率（0~1），本质是对 trace_id 空间做哈希分区，保证同一条 trace 在所有服务节点上的决策一致；限流采样则用令牌桶（token bucket）在客户端每秒放行固定数量的 span。自适应采样由 collector 周期性下发 SamplingStrategyResponse（per-operation 的采样率），客户端按 operation 名称缓存并套用哈希判断。决策结果写入 span context 的 flags 字节（bit0=sampled），通过 'uber-trace-id: {trace-id}:{span-id}:{parent-id}:{flags}' 或 W3C 'traceparent' 传播；子 span 的 tracer 读到 sampled=0 时直接不构造任何 span 数据，从源头截断。伪代码：

    func shouldSample(traceID, operation) bool {
        rate := getEffectiveRate(operation)  // 固定配置或远端下发的 param
        return hash(traceID) < rate * maxUint64
    }

尾部采样发生在收集端：collector 将到达的 span 按 traceID 分组存入内存缓冲，每个 trace 挂一个计时器（decision_wait，默认 10s）；计时器到期后按策略链依次求值——status_code 策略命中任意 ERROR span 则采，latency 策略命中端到端耗时阈值则采，probabilistic 策略对 traceID 再次做哈希判断；任一策略命中即把缓冲中的整组 span 写入存储，否则整体丢弃。其本质是用『延迟 + 内存』换取『基于事实的决策』，决策质量从先验概率升级为后验观测。

降采样的本质是闭环控制：collector 统计上一时间窗口内每个 operation 的 span 量与总 QPS，按存储预算反推各 operation 的目标采样率并下发；客户端下一窗口即按新 rate 采样。若在收集端二次降采样，必须按 traceID 整体丢弃，绝不能对 span 独立丢弃，否则破坏 trace 完整性并引入长度偏差。

与前端知识体系的对照：头部采样相当于在 React 事件处理函数入口处执行 Math.random() < 0.1 决定是否上报埋点——决策发生在结果产生之前，成本极低但看不见错误与延迟；尾部采样相当于 React Profiler 的 onRender 回调——组件提交后测量实际渲染耗时再决定是否记录，决策基于事实但需要额外缓冲与延迟；降采样相当于 requestAnimationFrame 在帧率超限时跳帧，或媒体流的动态码率控制——以保真度换取吞吐上限。更精确地说，头部采样是『声明式/编译期』决策（像 Java 接口：类声明时即确定契约，运行时不可变），尾部采样是『结构化/运行时』决策（像 TS 接口：使用时按实际形状结构匹配，延迟到最后一刻判定），降采样则是运行时的自适应契约调整。

### 3. 基础代码与实战验证
```text
下面用极简的 Python（jaeger-client）与 YAML（OpenTelemetry Collector）分别展示三种机制的落地形态。

1) 头部采样：客户端概率采样与限流采样

    from jaeger_client import Config

    # 头部采样决策在 tracer 初始化时确定，作用于每一个 trace 的根 span
    config = Config(
        config={
            'sampler': {
                'type': 'probabilistic',   # 按 trace_id 哈希分区，确定性决策
                'param': 0.1,              # 10% 的 trace 被采集
            },
            'logging': True,
        },
        service_name='order-svc',
    )
    tracer = config.initialize_tracer()

    # 创建根 span 时，sampler.should_sample() 被执行一次
    with tracer.start_span('create_order') as span:
        # sampled=1 的 trace 会为每个下游请求注入 uber-trace-id 头
        # sampled=0 的 trace 直接丢弃本 span，且子服务同样不采集
        span.set_tag('user.id', '12345')
    tracer.close()

    # 限流采样：type='ratelimiting', param=10 表示每秒最多采集 10 个 span
    # 底层是令牌桶，突发流量会被削峰，适用于高 QPS 且对精确率不敏感的场景

2) 尾部采样：collector 端按完整链路特征决策

    # jaeger-collector 或 OpenTelemetry Collector 的 tail_sampling 处理器
    processors:
      tail_sampling:
        decision_wait: 10s              # 缓冲窗口：首个 span 到达后等待 10 秒再决策
        num_traces: 50000               # 缓冲上限，超出后按最老 trace 优先淘汰
        policies:
          - name: error-policy
            type: status_code
            status_code: {status_codes: [ERROR]}   # 含错误 span 的 trace 必采
          - name: latency-policy
            type: latency
            latency: {threshold_ms: 500}           # 端到端延迟 > 500ms 必采
          - name: random-policy
            type: probabilistic
            probabilistic: {sampling_percentage: 10}  # 其余按 10% 兜底

    # 运行机制：span 先进入内存缓冲，10 秒窗口结束后依次求值策略链
    # 任一命中则整条 trace 写入存储；全部未命中则整条丢弃
    # 由此可知尾部采样无法节省客户端开销，只能削减存储成本

3) 降采样：自适应采样率的闭环

    # collector 端配置采样策略接口，按 QPS 动态下发
    # 伪代码：
    #   previous_window_span_count = sum(operation_span_counts)
    #   budget_per_window = 100000                       # 存储预算
    #   for op in operations:
    #       op.rate = budget_per_window * op.span_count / previous_window_span_count
    #   response = SamplingStrategyResponse(per_operation_rates=...)
    # 客户端收到后更新本地缓存，下一窗口按新 rate 执行哈希采样
    # 注意：降采样率必须整体作用于 trace，不能按 span 独立降采样
```

### 4. 常见误区与进阶思考
误区一：把『采样率 p』理解为『p% 的 span 被采集』。头部采样的单位是 trace 而非 span：一条 trace 要么全部 span 入库、要么一个都不入库。若在客户端错误实现为『每个 span 独立以概率 p 采样』，则含 N 个 span 的 trace 被观测到的概率为 1-(1-p)^N，span 越多的 trace 越容易被观测，导致从采集数据估计的 trace 长度分布发生上偏，延迟百分位与错误率随之失真。这正是头部采样必须把决策放在根 span 并随 context 传播的根本原因。另一种相关误区是在收集端做『按 span 降采样』来削减存储，这会以同样方式破坏 trace 完整性，使查询端无法重建调用链。

误区二：认为尾部采样可以替代头部采样、或能节省客户端开销。尾部采样只发生在 span 到达收集端之后：若头部采样率过低，错误或慢 trace 在客户端就被丢弃，尾部策略无从判定；反之若头部设为全量上报，尾部采样的缓冲窗口（如 10s）在峰值流量下会消耗巨额内存（内存 ≈ span 到达速率 × decision_wait × 平均 span 字节数），且所有 span 的入库延迟都被拉高到 decision_wait 量级。工程上通常组合使用：头部用高概率采样保证错误/慢链路完整到达，尾部用策略链精确筛选，降采样用于削峰与预算控制。

思考题：某服务产生 1000 trace/s，trace 长度 N 均值 5（几何分布）。你本应实现『整条 trace 以 p=0.1 概率采样』的头部采样，却误实现为『每个 span 独立以 p=0.1 采样』。请推导：(a) 两种实现下收集端收到的 span 速率是否相同？(b) 在错误实现下，被观测到的 trace（至少含 1 个采样 span）中 N 的条件均值是大于、等于还是小于 5？请用观测概率 1-(1-p)^N 解释。(c) 若真实错误率为 1% 且错误集中在长 trace，基于错误采样数据统计出的『错误 trace 占比』会高估还是低估？为什么？——本质是：采样单位从 trace 变成 span 后，观测概率与 trace 长度耦合，产生长度偏差（length bias）；头部采样正是通过『根 span 一次性决策 + context 传播』切断这种耦合。
