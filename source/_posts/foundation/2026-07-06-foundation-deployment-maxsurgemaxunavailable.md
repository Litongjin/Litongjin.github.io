---
title: "每日基础技术总结 · 2026-07-06 · Deployment 滚动更新的 maxSurge/maxUnavailable 计算与版本推进"
date: 2026-07-06 08:00:00
categories: [技术分享]
tags: ["技术分享", "DevOps 与云原生"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-06 · Deployment 滚动更新的 maxSurge/maxUnavailable 计算与版本推进

## 📚 今日主题

> **Deployment 滚动更新的 maxSurge/maxUnavailable 计算与版本推进**（DevOps 与云原生）

### 1. 核心概念速览
Deployment 滚动更新是 Kubernetes 中实现无中断版本发布的核心机制，其本质是控制器（Deployment Controller）基于声明式期望状态（desired state）驱动 ReplicaSet 数量调整的收敛过程。maxSurge 与 maxUnavailable 共同定义了滚动更新过程中 Pod 实例数量的允许偏移区间，即“期望副本数”的可接受范围。maxSurge 表示相对于期望副本数，允许额外多出的 Pod 数量（百分比或绝对数）；maxUnavailable 表示相对于期望副本数，允许不可用的 Pod 数量（百分比或绝对数）。二者共同约束了滚动更新期间的总 Pod 数上限（replicas + maxSurge）和可用 Pod 数下限（replicas - maxUnavailable）。该机制解决的核心问题是在不牺牲可用性（可用 Pod 数不低于阈值）和不超出资源配额（总 Pod 数不高于阈值）的前提下，逐步用新 ReplicaSet 替换旧 ReplicaSet。它在整个云原生体系中的位置是 Kubernetes 工作负载管理（Workload Management）的核心语义，直接关系到发布策略、故障恢复、资源弹性和系统稳定性。专业工程师必须掌握其精确计算方式，因为任何错误的配置都会导致发布阻塞、资源过载或服务不可用，且在生产环境中难以直观排查。

### 2. 底层原理剖析
底层机制由 Deployment Controller 中的 syncDeployment 循环驱动，核心是计算每个 ReplicaSet 的 desiredReplicas 和实际副本数，并通过 scale 子资源调整。滚动更新本质上是一个双指针收敛过程：旧 ReplicaSet 从 replicas 缩减，新 ReplicaSet 从 0 扩展，每一步都满足 maxSurge 和 maxUnavailable 的约束。计算逻辑如下：

1. 输入：replicas（期望副本数）、maxSurge（绝对数或百分比）、maxUnavailable（绝对数或百分比）、当前所有 ReplicaSet 的副本状态（availableReplicas、totalReplicas）。
2. 首先将百分比转换为绝对数（向下取整），maxSurge = ceil(replicas * surgePercent)? 实际Kubernetes中采用向上取整（ceil）计算 maxSurge 绝对值，而 maxUnavailable 向下取整（floor）。具体规则：maxSurge 绝对值 = ceil(replicas * surgePercent)，maxUnavailable 绝对值 = floor(replicas * unavailablePercent)。绝对数直接使用。
3. 计算当前总 Pod 数 total = 所有 RS 的副本数之和；当前可用 Pod 数 available = 所有 RS 的 availableReplicas 之和。
4. 滚动更新的目标状态是：新 RS 的副本数最终等于 replicas，旧 RS 副本数为 0。每一轮迭代的约束为：
   - 总副本数上限：total <= replicas + maxSurge
   - 可用副本数下限：available >= replicas - maxUnavailable
5. 在每次同步中，控制器优先扩新 RS，再缩旧 RS，顺序由 Deployment 的 strategy.RollingUpdate 配置决定。扩缩步长由上述约束反推：
   - 新 RS 可扩的最大数量 = min(replicas + maxSurge - total, replicas - newRS.currentReplicas, 其他约束)
   - 旧 RS 可缩的最大数量 = min(oldRS.currentReplicas - (replicas - maxUnavailable - available + oldRS.available? 实际逻辑更复杂)，但本质是保证 available 不低于下限。

更精确的伪代码（Kubernetes deployment_util.go 中的 ReplicaSetScaleUp/Down 逻辑简化）：

```
// 假设所有 RS 按版本新旧排列，currentRS 为最新 RS，oldRSs 为旧集合
allRSs := [old1, old2, ..., new]
totalReplicas := sum(allRSs.Replicas)
availableReplicas := sum(allRSs.AvailableReplicas)

// 计算总副本数上限
totalLimit := replicas + maxSurge
// 计算最小可用数
minAvailable := replicas - maxUnavailable

// 第一阶段：扩展新 RS
// 目标新 RS 副本数 = replicas，但受总上限和 minAvailable 影响
// 允许增加的总量 = totalLimit - totalReplicas
// 如果新 RS 当前副本 < replicas，则尝试扩展，但扩展后的 total 不能超过 totalLimit

// 第二阶段：缩减旧 RS
// 当新 RS 已经达到 replicas 或旧 RS 中的 Pod 已经 ready 后，开始缩减旧 RS
// 允许缩减的条件：缩减后 availableReplicas >= minAvailable 且 totalReplicas <= totalLimit
// 每次缩减一个 RS 的副本数，直到该 RS 为 0
```

关键点：maxSurge 控制“多快”能创建新 Pod，maxUnavailable 控制“多快”能删除旧 Pod。如果 maxUnavailable=0，则不允许任何 Pod 不可用，必须先创建新 Pod（需要 maxSurge>0），等待其 Ready 后才能删除旧 Pod。如果 maxSurge=0，则不允许超出总副本数，必须先删除旧 Pod（需要 maxUnavailable>0），才能创建新 Pod。两者通常需要配合使用，不能同时为 0（否则发布无法推进）。

与前端知识体系的对比：Deployment 滚动更新类似于前端状态管理中的“有限状态机”与“事务性更新”。前端的 React 状态批处理（Batching）将多次 setState 合并为一次渲染，而滚动更新则是将多个 scale 操作合并为一个个满足约束的中间状态。更贴近的是“乐观更新与回滚”机制：maxSurge 相当于乐观更新中允许的临时多余请求（pending 状态），maxUnavailable 相当于允许的降级响应（stale 数据）。但本质差异在于：前端状态更新通常是最终一致的、无严格资源边界，而 Kubernetes 的滚动更新是强约束的资源调度过程，每个中间状态都必须满足硬性资源和服务可用性条件。另一个对比：它类似前端的 A/B 测试中的流量切换比例，但这里的比例是 Pod 实例数量而非请求流量，且受可用性约束驱动。

### 3. 基础代码与实战验证
这里无法直接运行 Kubernetes 控制器的完整代码，但可以通过一个极简的 Python 模拟器来验证滚动更新的计算逻辑。以下代码不依赖任何框架，纯粹用状态变量模拟 maxSurge/maxUnavailable 的推进过程。

```python
# 滚动更新模拟器：验证 maxSurge/maxUnavailable 约束下的版本推进
def rolling_update(replicas, max_surge, max_unavailable, old_pods, new_pods):
    """
    replicas: 期望副本数
    max_surge: 绝对数（示例中直接给绝对数，实际K8s支持百分比）
    max_unavailable: 绝对数
    old_pods: 旧RS当前副本数（每个Pod初始为可用）
    new_pods: 新RS当前副本数
    返回每一步的状态列表
    """
    total_limit = replicas + max_surge  # 总副本数上限
    min_available = replicas - max_unavailable  # 可用Pod数下限
    history = []

    # 模拟Pod从旧RS迁移到新RS的过程
    while old_pods > 0 or new_pods < replicas:
        total = old_pods + new_pods
        # 计算可用Pod数：这里假设新Pod创建后立即Ready（真实情况需要等待健康检查）
        # 为了验证约束，我们假设所有Pod都可用
        available = old_pods + new_pods

        # 约束检查：如果当前状态不满足约束，则不能执行任何操作
        assert total <= total_limit, "超出总副本数上限"
        assert available >= min_available, "可用Pod数低于下限"

        # 记录当前状态
        history.append((old_pods, new_pods, total, available))

        # 策略：优先增加新RS，直到达到replicas或触达total_limit
        if new_pods < replicas and total < total_limit:
            new_pods += 1
        else:
            # 新RS已满或无法再扩，开始缩减旧RS，但要保证available >= min_available
            # 因为缩减旧RS会减少可用Pod数，所以需要预留足够的可用性
            if old_pods > 0 and (old_pods - 1 + new_pods) >= min_available:
                old_pods -= 1
            else:
                # 无法继续推进，说明maxSurge/maxUnavailable配置不合理
                print("更新无法继续，配置可能不满足推进条件")
                break

    # 最终状态
    history.append((old_pods, new_pods, old_pods + new_pods, old_pods + new_pods))
    return history

# 验证示例：replicas=3, maxSurge=1, maxUnavailable=0
# 初始：旧RS有3个Pod，新RS有0个
# 由于maxUnavailable=0，不允许减少可用Pod，只能先扩新Pod
history = rolling_update(replicas=3, max_surge=1, max_unavailable=0, old_pods=3, new_pods=0)
print("maxSurge=1, maxUnavailable=0:")
for step, (old, new, total, avail) in enumerate(history):
    print(f"Step {step}: old={old}, new={new}, total={total}, available={avail}")

# 输出应为：
# Step 0: old=3, new=0, total=3, available=3
# Step 1: old=3, new=1, total=4, available=4  （扩新，total达到limit=4）
# 然后开始缩旧，但每次缩旧后available要>=3
# Step 2: old=2, new=1, total=3, available=3
# Step 3: old=2, new=2, total=4, available=4  （扩新）
# Step 4: old=1, new=2, total=3, available=3
# Step 5: old=1, new=3, total=4, available=4
# Step 6: old=0, new=3, total=3, available=3

# 另一个极端：maxSurge=0, maxUnavailable=1
history2 = rolling_update(replicas=3, max_surge=0, max_unavailable=1, old_pods=3, new_pods=0)
print("\nmaxSurge=0, maxUnavailable=1:")
for step, (old, new, total, avail) in enumerate(history2):
    print(f"Step {step}: old={old}, new={new}, total={total}, available={avail}")
# 输出：先缩旧（因为total不能超过3），每次缩旧后available>=2
# Step 0: old=3, new=0, total=3, available=3
# Step 1: old=2, new=0, total=2, available=2  （缩旧，available=2=min_available）
# Step 2: old=2, new=1, total=3, available=3  （扩新）
# ... 循环直到完成
```

注释：代码中的 `assert` 验证了滚动更新每一步必须满足 `total <= replicas + maxSurge` 和 `available >= replicas - maxUnavailable`。实际 Kubernetes 控制器中，新 Pod 并非立即 Available，需要等待 Ready 条件；但计算约束的本质完全相同。通过调整 max_surge 和 max_unavailable，可以观察发布速度与可用性的权衡：maxSurge 越大，能越快创建新 Pod（但占用资源）；maxUnavailable 越大，能越快删除旧 Pod（但可能短暂降低容量）。

### 4. 常见误区与进阶思考
误区一：将 maxSurge 和 maxUnavailable 理解为“额外创建/删除的百分比”直接基于当前动态数量计算，而非基于期望副本数（replicas）计算。实际上，Kubernetes 在滚动更新开始时一次性将百分比转换为绝对数，并在整个更新过程中保持不变。例如 replicas=10，maxSurge=30%，maxUnavailable=30%，则 maxSurge=ceil(10*0.3)=4，maxUnavailable=floor(10*0.3)=3，而非每次调整时重新计算 30%。如果认为比例是动态的，会导致对中间状态总数估计错误，进而可能错误判断资源上限和可用下限。

误区二：认为 maxSurge=0 和 maxUnavailable=0 可以同时设置（或者认为这样会导致发布完全阻塞）。实际上，两者同时为 0 时，Deployment 无法进行任何 Pod 的增删，因为扩新会超出总副本数上限（replicas+0），缩旧会低于可用副本数下限（replicas-0）。Kubernetes 允许配置，但更新会一直停滞，直到用户修改配置。专业工程师必须理解这种配置的含义是“完全禁止任何变更”，而不是“快速更新”。

思考题：给定 replicas=5，maxSurge=1，maxUnavailable=1，初始旧 RS 有 5 个可用 Pod。如果新 Pod 创建后需要 30 秒才能进入 Available 状态，那么在滚动更新过程中，总 Pod 数和可用 Pod 数的变化曲线是什么？具体到每一步：旧 RS 从 5 缩到多少时，新 RS 才会开始扩展？当新 RS 有 2 个 Pod 但尚未 Available 时，旧 RS 能否继续缩减？请基于约束条件推导出所有可能的状态序列，并说明为什么某些看似合理的序列（例如先缩 2 个旧 Pod 再扩 2 个新 Pod）是合法的，而另一些序列（例如先缩 3 个旧 Pod 再扩）是非法的。这个思考题检验的是你是否真正理解 maxSurge/maxUnavailable 是全局约束而非局部操作许可。
