---
title: "每日基础技术总结 · 2026-07-02 · 连续批处理 Continuous Batching 与动态批处理"
date: 2026-07-02 08:00:00
categories: [技术分享]
tags: ["技术分享", "AI 开发基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-07-02 · 连续批处理 Continuous Batching 与动态批处理

## 📚 今日主题

> **连续批处理 Continuous Batching 与动态批处理**（AI 开发基础）

### 1. 核心概念速览
连续批处理（Continuous Batching）是一种动态调度推理请求的机制，其核心思想是：不再将多个请求固定为一个不可分割的批次（静态批处理），而是允许在每个模型前向传播步骤（step）后，动态地完成已完成请求的退出、插入新请求，并保持剩余请求的KV Cache与计算流形持续有效。本质上是将『批处理』从请求级粒度细化到步骤级粒度，将GPU的计算资源从『按请求批次分配』变为『按计算步骤动态复用』。它解决的核心问题是：静态批处理中，短请求被长请求阻塞导致GPU空闲（即批间等待），以及新请求必须等待整个批次完成才能进入的延迟问题。动态批处理（Dynamic Batching）是连续批处理的泛化形式，它允许在运行时根据实时负载、请求长度、显存余量等因素动态调整批大小和组成，而连续批处理是其中一种最典型的实现策略。该机制位于AI推理系统架构的调度层，介于上层的推理服务框架（如vLLM、TensorRT-LLM）与下层的GPU执行内核之间，是推理引擎性能优化的核心杠杆。专业工程师必须掌握它，因为它直接决定了推理服务的吞吐量（Tokens/s）和延迟分位数（TTFT/TPOT），是生产环境LLM部署与性能调优的基础认知，也是理解现代推理框架源码的钥匙。

### 2. 底层原理剖析
底层运行机制的本质是：将模型推理拆解为多个连续的步骤（通常每个步骤处理一个token的解码），并在每个步骤开始时，重新计算当前批次的有效请求集合。关键数据结构是每个请求的KV Cache（键值缓存）与调度状态机。

伪代码如下：

```
while True:
    # 1. 收集新到达请求，判断显存是否足够分配KV Cache
    for req in incoming_requests:
        if can_allocate(req):
            allocate_kv_cache(req)
            req.state = RUNNING
            add_to_active_set(req)

    # 2. 遍历活跃请求，过滤已完成（生成EOS或达到max_len）的请求
    active_requests = [req for req in active_requests if not req.is_finished]

    # 3. 对每个活跃请求，取当前需解码的token，拼成一个batch
    batch_input = [req.next_token_id for req in active_requests]

    # 4. 执行一次前向传播，得到每个请求的logits
    logits = model_forward(batch_input, kv_cache_indices)

    # 5. 分别采样、更新每个请求的状态与KV Cache
    for i, req in enumerate(active_requests):
        req.next_token_id = sample(logits[i])
        update_kv_cache(req, req.next_token_id)
        if req.is_finished:
            req.state = DONE
            free_kv_cache(req)
```

关键机制要点：
- 每步前向传播的批次大小和成员是可变的，GPU线程束（warp）不会因为等待最慢请求而空转——因为每个请求只推进一个token，天然对齐了计算粒度。
- KV Cache的预分配与显存管理是连续批处理能否高效运行的前提，通常采用分页（PagedAttention）或块分配策略，避免碎片化。
- 调度器需要在每一步决定是否插入新请求（受限于显存预算和延迟目标），这就是动态性的来源。

与前端已有知识体系的对比：这类似前端事件循环中的微任务批处理——每个宏任务（一次前向传播）内，微任务队列（活跃请求）是动态变化的，可以不断追加新微任务（新请求），并且已完成的微任务会被移除。但更精确的类比是浏览器渲染管线中的增量布局：每次只处理一个变更，并动态合并计算。而JavaScript的Promise批处理（Promise.all）则对应静态批处理，必须等待所有Promise完成。另一个类比是React的并发调度（Concurrent Mode）中的可中断渲染：将渲染拆分为多个时间片，每个时间片处理一部分，并在时间片之间响应更高优先级更新，这与连续批处理将推理拆分为多步并在步骤间插入新请求有同构性。但关键区别在于：前端调度是单线程协作式，而连续批处理是在GPU并行硬件上追求吞吐最大化的细粒度流水线。

### 3. 基础代码与实战验证
以下是基于Python伪代码实现的最小化连续批处理调度器，不依赖任何推理框架，仅演示核心调度逻辑（假设已具备模型前向函数model_forward和KV Cache管理器）。

```python
class ContinuousBatchScheduler:
    def __init__(self, model, max_batch_size, kv_cache_manager):
        self.model = model
        self.max_batch_size = max_batch_size
        self.kv_cache_manager = kv_cache_manager
        self.active_requests = []  # 当前活跃请求列表，每个请求含input_ids, kv_cache, state

    def step(self, new_requests):
        # 1. 尝试将新请求加入活跃集合，前提是显存足够且不超过最大batch
        for req in new_requests:
            if len(self.active_requests) < self.max_batch_size and \
               self.kv_cache_manager.can_allocate(req):
                req.kv_cache = self.kv_cache_manager.allocate(req)
                req.state = 'RUNNING'
                self.active_requests.append(req)

        # 2. 过滤已完成请求（生成EOS或达到长度限制），释放其KV Cache
        still_active = []
        for req in self.active_requests:
            if req.state == 'FINISHED':
                self.kv_cache_manager.free(req.kv_cache)
            else:
                still_active.append(req)
        self.active_requests = still_active

        # 3. 若没有活跃请求，直接返回
        if not self.active_requests:
            return

        # 4. 构造当前步骤的输入：每个活跃请求当前最后生成的token id
        input_tokens = [req.input_ids[-1] for req in self.active_requests]

        # 5. 执行一次前向传播（这里仅模拟返回logits，实际中会计算attention并更新KV Cache）
        logits_list = self.model.forward(input_tokens, [req.kv_cache for req in self.active_requests])

        # 6. 对每个请求独立采样并更新状态
        for i, req in enumerate(self.active_requests):
            next_token = sample_from_logits(logits_list[i])
            req.input_ids.append(next_token)
            self.kv_cache_manager.update(req.kv_cache, next_token)  # 将新token写入KV Cache
            if next_token == EOS_TOKEN_ID or len(req.input_ids) >= req.max_len:
                req.state = 'FINISHED'
```

关键行注释：
- `input_tokens = [req.input_ids[-1] for req in self.active_requests]`：每步只取每个请求的最新token，这保证了所有请求的序列长度对齐，避免padding浪费。
- `self.model.forward(input_tokens, ...)`：一次前向传播同时处理多个序列，但每个序列只生成一个token，这正是连续批处理的核心——计算步骤级动态合并。
- `if next_token == EOS_TOKEN_ID ...`：完成判断发生在步骤内部，完成后立即在下一轮step开始时释放资源，而不是等待整个批次结束。

运行循环：
```python
scheduler = ContinuousBatchScheduler(...)
while True:
    new_requests = receive_requests()  # 从队列获取新请求
    scheduler.step(new_requests)
    output_requests = [req for req in scheduler.active_requests if req.state == 'FINISHED']
    # 将完成结果返回给客户端
```

该代码展示了动态批处理的最小闭环：动态加入、动态退出、步级合并。实际生产系统会在此基础上增加优先级调度、抢占、投机采样等优化。

### 4. 常见误区与进阶思考
误区一：认为连续批处理就是增大batch size。实际上，静态批处理也能用大batch size，但问题在于batch中所有请求必须同时开始、同时结束，导致GPU在批次末尾出现严重空闲（straggler效应）。连续批处理的本质不是增大batch，而是通过细粒度调度消除空闲气泡，因此它的优势在高请求到达方差和长度差异大的场景下尤为明显，而大batch size只是其可能带来的结果而非原因。

误区二：混淆连续批处理与动态批处理（Dynamic Batching）为同一概念。动态批处理是一个更宽泛的范畴，它指任何在运行时根据实时状态调整批处理组成的策略，例如基于请求队列长度或GPU负载的贪心合并、超时批处理等。连续批处理是动态批处理的一种特定实现，强调在每个decode步骤之间进行调度决策，而非仅在新请求到达或批处理完成时调整。理解这一区别有助于正确阅读框架源码和设计调度器。

思考题：假设一个静态批处理系统，批次大小为N，每个请求的生成长度服从指数分布，平均长度为L。当N增大时，该批次的完成时间由最长的请求决定，因此系统吞吐量随N增长出现什么变化？请从GPU利用率的角度推导：在连续批处理下，为什么同样N值对应的吞吐量会显著高于静态批处理？请给出具体的时间线对比分析。回答此题需理解：静态批处理中，每步前向传播计算量固定为N个序列，但部分序列提前结束后其计算槽位被浪费；而连续批处理能在该槽位立即插入新序列，从而在相同总前向传播步数内完成更多请求，这正是KV Cache复用和显存管理带来的收益。若你能定量解释这一点，即真正理解了连续批处理的底层价值。
