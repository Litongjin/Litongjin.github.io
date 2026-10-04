---
title: "每日基础技术总结 · 2026-10-05 · Deployment 滚动更新与回滚策略"
date: 2026-10-05 07:05:22
categories: [技术分享]
tags: ["技术分享", "云原生与 DevOps（Docker / K8s）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-05 · Deployment 滚动更新与回滚策略

## 📚 今日主题

> **Deployment 滚动更新与回滚策略**（云原生与 DevOps（Docker / K8s））

### 1. 核心概念速览
Deployment 是 Kubernetes 中声明式无状态应用发布控制器，滚动更新与回滚是其内置的发布安全机制。滚动更新通过渐进式替换旧 Pod 为新 Pod，在更新过程中维持服务可用性；回滚则通过切换 ReplicaSet 版本引用，将 Deployment 重新指向历史稳定副本集。其本质是控制器模式 + 声明式期望状态收敛 + 版本化 ReplicaSet 管理的组合结果，而不是简单的“重启容器”。它解决的核心问题是：在分布式环境下，应用变更必然发生，如何在变更过程中控制故障半径、保持服务连续性，并具备可审计、可撤销的发布能力。
在云原生与 DevOps 体系中，Deployment 的滚动更新与回滚位于“应用交付层”和“运行时稳定性控制层”之间。它向下依赖 Pod、ReplicaSet、kube-scheduler、kubelet、CNI、Service Endpoint 机制，向上承接 CI/CD 流水线、灰度发布、金丝雀发布、SLO 保障。对于从前端转向后端与 AI 基础设施的工程师而言，这是理解“不可变基础设施”“声明式系统”“控制循环”“发布工程”的最小但关键入口。专业工程师必须掌握它，因为任何生产系统都无法回避版本发布、故障恢复、依赖升级、配置变更与流量切换，而这些能力最终都会落到控制器如何安全地从当前状态收敛到目标状态。

### 2. 底层原理剖析
Deployment 的核心不是直接管理 Pod，而是管理 ReplicaSet。Deployment 定义期望模板，Deployment Controller 监听 Deployment、ReplicaSet、Pod 的变化，通过控制循环不断将实际状态向期望状态推进。每次更新 PodTemplate 会生成新的 ReplicaSet，Deployment 将流量目标逐步切换到新 ReplicaSet，同时按比例缩减旧 ReplicaSet。

滚动更新的关键参数有两个：maxSurge 与 maxUnavailable。maxSurge 控制更新期间允许超过期望副本数的额外 Pod 数量，用于先扩容新版本；maxUnavailable 控制更新期间允许不可用 Pod 的最大数量，用于限制服务容量下降幅度。二者共同决定更新速度、资源占用与可用性边界。更新过程中，新 Pod 不会在创建后立即承接流量，必须通过 readiness probe 进入 Ready 状态，才会被 EndpointSlice/Endpoints 纳入 Service 后端。旧 Pod 也不会立即删除，而是经历 terminating、preStop、SIGTERM、grace period 的优雅退出流程。

伪代码：

WHEN deployment.spec.template CHANGES:
  newRS = CREATE_REPLICASET(hash(deployment.spec.template))
  oldRSList = LIST_REPLICASET_EXCLUDING(newRS)
  WHILE newRS.replicas < deployment.spec.replicas:
    INCREASE newRS.replicas BY maxSurge
    WAIT_FOR newPods READY
    DECREASE oldRS.replicas BY maxUnavailable
  WHEN newRS READY AND oldRS SCALED_TO_ZERO:
    MARK deployment AVAILABLE
    RECORD revision history

回滚的本质不是重新构建镜像，也不是删除新 ReplicaSet，而是将 Deployment 的 PodTemplate 恢复到历史 revision 对应的模板。Kubernetes 会基于历史 ReplicaSet 或 Deployment revision 生成新的目标状态，控制器再次执行收敛流程，将副本从当前版本迁移到旧版本。若历史 ReplicaSet 仍存在，则回滚可以非常快；若依赖外部镜像仓库或构建链路，回滚速度会受拉取镜像、调度、探针通过时间影响。

与前端已有知识对比：前端常见“热更新”或模块替换关注的是运行时内存中的模块图更新，通常不改变进程生命周期；Deployment 滚动更新则是进程级、容器级、节点级的替换，属于不可变基础设施。它不是修改正在运行的容器，而是创建新容器、停止旧容器。类比 TS/Java 接口：TS 接口主要是编译期类型约束，运行时不保留结构；Java 接口是运行时类型系统的一部分，可被反射、动态代理、JVM 类型检查使用。Deployment 的声明式 spec 更像运行时契约：它不只是类型声明，而是被控制器持续执行的状态目标。TS 接口描述“代码应该怎样”，Deployment spec 描述“系统当前应该变成什么”，控制器负责持续逼近该目标。

### 3. 基础代码与实战验证
```text
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 4
  revisionHistoryLimit: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      terminationGracePeriodSeconds: 30
      containers:
      - name: web
        image: web:1.0.0
        readinessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 2
          periodSeconds: 2
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 5"]

kubectl set image deployment/web web=web:1.1.0
kubectl rollout status deployment/web
kubectl rollout history deployment/web
kubectl rollout undo deployment/web --to-revision=2

说明：strategy.type 指定发布算法为滚动更新，而不是 Recreate。maxSurge=1 表示更新期间最多允许 5 个 Pod 同时存在，用于提前扩容新版本。maxUnavailable=0 表示任何时刻都必须至少有 4 个可用副本，禁止容量低于期望值。readinessProbe 决定新 Pod 是否能进入 Service 后端，未 Ready 前不会承接流量，这是滚动更新可用性的关键。terminationGracePeriodSeconds 定义 Pod 删除后的优雅退出总窗口。preStop sleep 用于延缓容器退出，给 kube-proxy、EndpointSlice、负载均衡、连接排空留出传播时间，避免旧连接在路由尚未更新时被强制切断。kubectl set image 会修改 Deployment 的 PodTemplate，触发新 ReplicaSet 创建。kubectl rollout status 是观察控制器收敛过程的命令。kubectl rollout history 展示 Deployment 的版本历史，每个 revision 对应一次模板变更。kubectl rollout undo 并非简单删除新 Pod，而是让 Deployment 重新声明旧模板，控制器再次执行滚动收敛，将副本迁移回旧 ReplicaSet。
```

### 4. 常见误区与进阶思考
误区一：认为滚动更新等于零停机。滚动更新只是控制 Pod 替换节奏，真正的零停机还依赖镜像拉取成功、readiness probe 准确、应用支持优雅关闭、长连接正确排空、Service 与负载均衡路由收敛、依赖服务兼容。若应用启动后立即崩溃、探针过宽、数据库迁移不兼容、旧连接被强制关闭，仍然会出现故障。
误区二：认为回滚就是回到上一个镜像版本。Deployment 回滚回滚的是 PodTemplate 与 revision 状态，包括镜像、环境变量、资源限制、探针、label 等。若新版本涉及不兼容数据库 schema、配置中心状态或外部依赖变更，单纯回滚 Deployment 可能无法恢复业务一致性，甚至导致旧版本无法连接新数据格式。
进阶思考题：如果 maxSurge=0、maxUnavailable=1，同时新 Pod 的 readiness probe 一直失败，Deployment 控制器、ReplicaSet、旧 Pod、Service Endpoints 会分别处于什么状态？为什么此时系统不会完成回滚，也不会自动认为新版本失败？这反映了 Kubernetes 声明式控制系统的什么边界？
