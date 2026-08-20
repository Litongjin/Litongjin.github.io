---
title: "每日基础技术总结 · 2026-07-06 · Kubernetes 的 pause 容器与 Pod 共享命名空间的生命周期"
date: 2026-07-06 08:00:00
categories: [技术分享]
tags: ["技术分享", "DevOps 与云原生"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-06 · Kubernetes 的 pause 容器与 Pod 共享命名空间的生命周期

## 📚 今日主题

> **Kubernetes 的 pause 容器与 Pod 共享命名空间的生命周期**（DevOps 与云原生）

### 1. 核心概念速览
Kubernetes 的 pause 容器（又称 infra container）是每个 Pod 中第一个被创建的容器，其唯一职责是作为 Pod 共享命名空间的锚点。Pod 的所有业务容器（包括 init 容器）通过 join 该容器的命名空间（PID、Network、IPC、UTS、Mount）来组成一个逻辑上的“单元”。本质上，pause 容器是 Pod 生命周期与命名空间生命周期的载体：当 kubelet 创建 Pod 时，先启动 pause 容器，再依次启动业务容器；当 pause 容器退出时，其持有的命名空间被内核回收，整个 Pod 随之终结。它解决的核心问题是：让业务容器的启动、退出、重启不影响 Pod 命名空间的稳定性，从而确保网络、进程视图、共享内存等资源在 Pod 内保持连续。在计算机体系结构中，它位于容器运行时（CRI）与 Linux 内核命名空间机制之间，是 Kubernetes 对 Linux 进程隔离的抽象封装。专业工程师必须掌握它，因为它是理解 Pod 生命周期、容器重启语义、网络模型（如 CNI 的 bridge 模式）、以及故障排查（如 nsenter 进入 Pod 网络空间）的基石；忽略 pause 容器将导致对 Pod 资源占用、进程管理、和调试方法论的错误认知。

### 2. 底层原理剖析
底层机制：kubelet 通过 CRI 调用容器运行时（containerd、CRI-O 等）创建 Pod 时，首先运行 pause 镜像的容器。pause 进程在启动时创建一个全新的命名空间集合（通过 unshare 或 clone 设置 CLONE_NEWPID、CLONE_NEWNET、CLONE_NEWIPC、CLONE_NEWUTS、CLONE_NEWNS 标志），然后进入休眠状态。随后，每个业务容器通过 setns(2) 系统调用加入 pause 容器对应的命名空间。关键点：业务容器不是自己创建命名空间，而是主动 attach 到已有命名空间。因此，pause 容器成为这些命名空间的“引用持有者”——只要 pause 进程存活，命名空间就不会被内核销毁；即使所有业务容器都退出，Pod 仍然存在，并且网络栈（如 IP 地址、路由、端口）保持不变。当 pause 容器被杀死（例如节点驱逐、手动删除 Pod），其文件描述符关闭，内核回收所有相关命名空间，Pod 中所有剩余业务容器被迫终止。

流程伪代码：
1. kubelet 调用 CRI CreateContainer(pause)
2. runc/containerd 创建 pause 进程，分配新的命名空间
3. kubelet 调用 CRI CreateContainer(biz-container-1)，运行时执行 setns(pause 的 pid, 所有命名空间)
4. 业务容器启动，其 PID 1 是业务进程，但它是 pause 的命名空间子进程
5. Pod 状态由 pause 容器主导；业务容器退出不影响命名空间
6. 删除 Pod 时，kubelet 先停止业务容器，最后停止 pause 容器，命名空间销毁

与前端已有概念的对比：这类似于浏览器标签页与 iframe 的关系——pause 容器相当于标签页的“根文档”，它创建并持有页面生命周期（命名空间），业务容器相当于内嵌的 worker 或子 iframe，它们的崩溃不会导致标签页关闭。更精确地类比：在 JavaScript 中，模块的顶层执行环境（global scope）由 JS 引擎持有，每个模块（业务容器）共享同一全局对象；若全局对象被回收（pause 退出），所有模块的作用域随之消失。此外，它类似于 TS 接口 vs Java 接口的本质差异：Java 接口是强约束的“契约”，而 TS 接口是结构化的“形态”；pause 容器是“物理的命名空间持有者”，而业务容器是“逻辑上的租户”，两者是所有权与借用关系，而非契约关系。

### 3. 基础代码与实战验证
```text
以下为在 Kubernetes 节点上验证 pause 容器与命名空间生命周期的精确步骤（使用命令行工具，不依赖任何框架）：

# 1. 创建一个简单的 Pod（如 nginx）并获取其名称
kubectl run test-pod --image=nginx --restart=Never

# 2. 在节点上找到该 Pod 的 pause 容器（通常名为 "pause" 或 "POD"），并获取其 PID
docker ps --filter "name=k8s_POD_test-pod_default" --format "{{.ID}} {{.Names}}"
# 或使用 crictl（containerd 环境）
crictl ps --name POD

# 3. 进入 pause 容器的命名空间，观察其网络与进程视图
nsenter -t <pause-pid> -n ip addr
# 输出将显示 Pod 的 IP 地址（例如 eth0@if...），这是业务容器共享的网络栈

# 4. 进入业务容器（nginx）的命名空间，与 pause 的命名空间对比
nsenter -t <nginx-pid> -n ip addr
# 输出完全一致，证明业务容器复用了 pause 的网络命名空间

# 5. 验证进程视图：pause 是 PID 1，nginx 进程是其子进程（在 PID 命名空间内）
nsenter -t <pause-pid> -p ps aux
# 可以看到 PID 1 是 /pause，PID 2 左右是 nginx worker 进程

# 6. 验证生命周期：手动 kill pause 进程（模拟崩溃），观察 Pod 状态
kill -9 <pause-pid>
# 随后 `kubectl get pod` 会看到该 Pod 处于 Error/Unknown 状态，且所有业务容器被终止
# 因为命名空间持有者消失，内核回收了所有资源

注释：第 3-4 步验证了命名空间的“共享”本质——网络接口、路由、ARP 表在 Pod 内完全一致；第 5 步验证了 PID 命名空间内进程树的父子关系；第 6 步验证了“pause 容器的生命周期即 Pod 生命周期”这一核心原则。
```

### 4. 常见误区与进阶思考
误区一：认为 pause 容器是可优化的“多余开销”，甚至尝试使用 --share-process-namespace=false 或自定义容器替代。实际上，pause 容器是 Kubernetes 架构的基石，它保证了 Pod 内所有容器共享命名空间的顺序性和稳定性。去掉 pause 容器会导致业务容器各自持有命名空间，Pod 将退化为一组松散容器，无法满足 Kubernetes 对 Pod 的原子性调度、网络模型和资源管理要求。

误区二：认为业务容器（如主应用）退出会导致 Pod 终止，从而影响整个命名空间。实际上，Pod 是否终止取决于 restartPolicy 和 pause 容器状态，而非业务容器退出。如果 restartPolicy=Always，业务容器退出后会被 kubelet 重启，而网络 IP、共享内存等状态因 pause 容器未退出而保持不变。反之，只有当 pause 容器退出，命名空间才会销毁，所有容器才会彻底终止。

思考题：当你在 Pod 中设置 hostNetwork: true 时，pause 容器的网络命名空间是如何被处理的？它是否还持有独立的网络命名空间，还是直接与宿主机共享？请从 Kubernetes 源码中 pause 容器的创建逻辑和 CRI 的 namespace 配置选项（如 LinuxContainerConfig.Namespaces）出发，解释此时 pause 容器的作用是否仍然成立，以及这对 Pod 内容器网络隔离有何影响。
