---
title: "每日基础技术总结 · 2024-06-24 · Pod 生命周期：Init/探针与重启策略"
date: 2024-06-24 20:00:00
categories: [技术分享]
tags: ["技术分享", "云原生与 DevOps（Docker / K8s）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-06-24 · Pod 生命周期：Init/探针与重启策略

## 📚 今日主题

> **Pod 生命周期：Init/探针与重启策略**（云原生与 DevOps（Docker / K8s））

### 1. 核心概念速览
Pod 生命周期管理是 Kubernetes 控制平面确保容器组状态一致性的核心机制，本质是通过声明式 API 将应用运行态的期望状态转化为控制回路的实际操作。Init Containers 解决依赖初始化与环境隔离问题，其严格串行执行语义与 Sidecar 模式分离关注点；Probe（Readiness/Liveness/Startup）通过 HTTP/TCP/gRPC 命令字轮询实现健康检测，解决服务发现路由准确性与故障自愈时效性问题；Restart Policy 定义容器崩溃后的生命周期终结策略，体现系统级容错设计。掌握此知识点是理解分布式系统最终一致性、无状态服务治理及混沌工程的基础，对于从前端转向后端/AI 部署的关键在于理解‘基础设施即代码’中对状态机的抽象与自动化维护。

### 2. 底层原理剖析
1. Init Container 机制：在主业务容器启动前按序执行，拥有独立的镜像挂载与网络命名空间。其 exit code 为 0 则继续，非 0 则重试（受 RestartPolicy 约束）。底层依赖 containerd/docker shim 的生命周期钩子。
2. Probe 机制：kubelet 周期性执行探测逻辑。
   - Startup Probe：阻塞 Readiness/Liveness 检查，适用于冷启动慢的应用。失败导致容器重启。
   - Liveness Probe：检测存活性。失败触发容器重启（OOM 除外），不阻断服务发布。
   - Readiness Probe：检测可用性。失败将 Pod IP 从 Service Endpoint 摘除，停止流量转发，但不重启容器。
   三者竞争关系由 spec.containers[*].startupProbe 的存在与否决定优先级队列。
3. Restart Policy：仅支持 Always（默认）、OnFailure、Never。应用于 Pod 内所有容器，任一容器退出且策略允许时，kubelet 重新创建整个 Pod 上下文（包括网络挂载和卷绑定）。

### 3. 基础代码与实战验证
```text
# Kubernetes Manifest (YAML) - 核心结构示意
apiVersion: v1
kind: Pod
metadata:
  name: complex-app
spec:
  restartPolicy: OnFailure # 仅在非成功终止时重启
  initContainers:
    - name: init-config
      image: busybox
      command: ['sh', '-c', 'until nslookup mydb; do echo waiting; sleep 2; done'] # 阻塞主容器直至依赖就绪
  containers:
    - name: app
      image: nginx
      startupProbe:
        tcpSocket: { port: 80 }       # 最长等待 30s (period*5)，期间不检查 liveness/readiness
        periodSeconds: 3
      livenessProbe:
        httpGet: { path: /healthz, port: 80 }
        failureThreshold: 3          # 连续3次失败才触发重启
      readinessProbe:
        httpGet: { path: /ready, port: 80 }
        initialDelaySeconds: 5       # 容器启动后延迟5s开始探测
```

### 4. 常见误区与进阶思考
误区 1：混淆 Liveness 与 Readiness 的职责。Liveness 用于修复死锁或无法恢复的状态，频繁触发会导致服务震荡（Restart Loop）；Readiness 用于负载均衡削峰填谷。若对静态资源服务启用 Liveness 且配置不当，会在重启瞬间造成请求丢失。误区 2：认为 Init Container 失败会自动进入 CrashLoopBackOff 而不受重启策略影响。实际上 Init 阶段失败遵循 Pod 级别的 RestartPolicy，通常表现为 Pending 状态而非 Running->Crash 状态。思考题：当 Startup Probe 设置过长导致服务不可用时间超过 SLA 时，如何通过架构调整平衡‘缓慢启动’与‘快速暴露’之间的矛盾？
