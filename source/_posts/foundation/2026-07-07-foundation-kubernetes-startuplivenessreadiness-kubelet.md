---
title: "每日基础技术总结 · 2026-07-07 · Kubernetes 探针的 startup/liveness/readiness 与 kubelet 的重启策略"
date: 2026-07-07 08:00:00
categories: [技术分享]
tags: ["技术分享", "DevOps 与云原生"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-07 · Kubernetes 探针的 startup/liveness/readiness 与 kubelet 的重启策略

## 📚 今日主题

> **Kubernetes 探针的 startup/liveness/readiness 与 kubelet 的重启策略**（DevOps 与云原生）

### 1. 核心概念速览
Kubernetes 探针是 kubelet 对容器进行周期性健康检查的机制，分为 startupProbe、livenessProbe、readinessProbe 三类，分别解决「容器是否已完成启动」「容器是否存活」「容器是否具备服务能力」三个正交问题。其本质是 kubelet 通过执行 HTTP GET、TCP Socket 或 Exec 命令获取容器状态，并据此驱动 Pod 生命周期动作：startup 探针成功前不启动其他探针，liveness 失败触发重启（由 restartPolicy 决定），readiness 失败则从 Service Endpoints 中摘除流量。在整个云原生体系中，探针是控制平面与数据平面之间的关键反馈回路，是声明式状态向实际运行状态收敛的传感器，也是实现自愈能力的基础原语。专业工程师必须掌握它，因为任何生产级工作负载的可用性保障、滚动发布策略、故障恢复 SLA 都直接依赖对这些机制的精确理解，而非表面配置。

### 2. 底层原理剖析
kubelet 是唯一执行探针的组件，它通过容器运行时（CRI）在容器内执行检查。每个探针具有三个核心参数：initialDelaySeconds（容器启动后等待多久开始探测）、periodSeconds（探测周期）、failureThreshold（连续失败多少次才判定为失败）。startupProbe 是门卫：若配置了 startupProbe，则 kubelet 在 startupProbe 成功之前不会启动 liveness/readiness 探测；这用于保护启动时间很长的容器（如 JVM 应用）不被 liveness 提前杀死。livenessProbe 失败后，kubelet 根据 Pod 的 restartPolicy 决定动作：restartPolicy=Always/OnFailure 时重启容器；Never 则不重启，容器保持终止状态。readinessProbe 失败不会触发重启，而是将 Pod 的 Ready 条件置为 False，Endpoint Controller 随后从所有关联 Service 的 Endpoints 列表中移除该 Pod IP。三者状态机可抽象为：StartupGate -> LivenessLoop -> ReadinessGate。底层实现上，kubelet 调用 runtime.ExecSync 或通过 HTTP 客户端/TCP 拨号进行探测，探测结果通过 ProbeManager 更新至 PodStatus，再经 kubelet 状态管理器同步到 API Server。与前端概念对比：readinessProbe 类似于前端路由中的健康检查（如 NGINX upstream 的 active check），决定流量是否分发；livenessProbe 类似于 Node.js 的 process 存活监控，但粒度更细；startupProbe 类似于前端构建中的「启动编译完成前不接入流量」的门槛，但它是基于显式检查而非超时。探针与 restartPolicy 的关系不同于 TypeScript 接口与 Java 接口——后者是编译期契约的差异，而探针是运行期行为策略的分离：restartPolicy 决定「死了怎么办」，liveness 决定「是否算死」，readiness 决定「是否算可用」，三者通过 kubelet 的状态机解耦，而非通过类型系统耦合。

### 3. 基础代码与实战验证
```text
以下为最小可验证的 Pod 配置（YAML 本质是 API 对象的声明式描述），展示三类探针与重启策略的协同：

apiVersion: v1
kind: Pod
metadata:
  name: probe-demo
spec:
  restartPolicy: Always   # 容器退出后总是重启，与 liveness 失败触发的重启一致
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "sleep 3 && touch /tmp/ready && sleep 3600"]
    startupProbe:
      exec:
        command: ["cat", "/tmp/startup"]  # 此文件永远不会创建，startupProbe 持续失败
      periodSeconds: 2
      failureThreshold: 3                  # 连续失败 3 次后，startup 判定为失败，Pod 进入 NotReady 并最终按策略重启？注意：startup 失败也会按 restartPolicy 重启容器
    livenessProbe:
      exec:
        command: ["cat", "/tmp/healthy"]  # 该文件不存在，一旦 startup 成功后 liveness 会立即失败
      initialDelaySeconds: 1
      periodSeconds: 2
      failureThreshold: 1                  # 1 次失败即重启容器
    readinessProbe:
      exec:
        command: ["cat", "/tmp/ready"]   # 容器启动 3 秒后创建该文件，readiness 从失败转为成功
      initialDelaySeconds: 1
      periodSeconds: 1
      failureThreshold: 1

关键执行逻辑（用伪代码描述 kubelet 内部探测循环）：

for each container in pod:
    if startupProbe configured:
        if not startupProbe.succeeded:
            result = runProbe(container.startupProbe)
            if result.success:
                startupProbe.succeeded = true
                startLivenessAndReadiness()
            else:
                startupProbe.failures++
                if startupProbe.failures >= failureThreshold:
                    markContainerFailed()   # 触发 restartPolicy 决策，通常重启容器
                continue  # 此时 liveness/readiness 不执行
    # startup 已成功，进入常规循环
    livenessResult = runProbe(container.livenessProbe)
    if livenessResult.failed:
        killContainerAndRestart()  # 依据 restartPolicy
    readinessResult = runProbe(container.readinessProbe)
    updatePodReadyCondition(readinessResult.success)  # 失败则从 Endpoints 摘除

注意：上述 startupProbe 命令故意永不成功，实际观察时容器会被无限重启（因为 restartPolicy=Always），而 readinessProbe 不会影响重启。若要验证正常流程，可将 startupProbe 命令改为 ["cat", "/tmp/ready"] 或删除 startupProbe。这段代码的最小性在于只用 busybox 内置命令，不依赖任何框架，直接验证探针语义。
```

### 4. 常见误区与进阶思考
误区一：认为 readinessProbe 失败会重启容器。实际上 readiness 失败只影响 Pod 的 Ready 状态和 Service 端点摘除，kubelet 不会因 readiness 失败触发任何容器重启。只有 liveness 失败（或 startup 失败）才会按 restartPolicy 重启。很多排障者看到 Pod 被重启，误以为是 readiness 导致，实际应检查 liveness 或 startup 配置。误区二：认为 startupProbe 成功后就再也不运行。startupProbe 只在启动阶段运行，成功后即被禁用，后续生命周期由 liveness/readiness 接管。将启动检查逻辑放在 liveness 中且未设置足够大的 initialDelaySeconds，是导致启动慢的容器被反复杀死的常见原因。

思考题：在 Pod 的 restartPolicy=Never 且 livenessProbe 连续失败的情况下，kubelet 会终止容器，但 Pod 停留在什么 phase？该 Pod 是否会被从 Service Endpoints 中移除？请从探针状态更新与 Endpoint Controller 的 watch 机制推导完整链路。
