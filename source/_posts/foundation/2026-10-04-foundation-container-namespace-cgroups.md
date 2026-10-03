---
title: "每日基础技术总结 · 2026-10-04 · 容器隔离：Namespace 与 Cgroups 原理"
date: 2026-10-04 07:11:14
categories: [技术分享]
tags: ["技术分享", "云原生与 DevOps（Docker / K8s）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-04 · 容器隔离：Namespace 与 Cgroups 原理

## 📚 今日主题

> **容器隔离：Namespace 与 Cgroups 原理**（云原生与 DevOps（Docker / K8s））

### 1. 核心概念速览
容器隔离的本质是 Linux 内核提供的两类进程约束机制：Namespaces 负责视图隔离，Cgroups 负责资源约束。二者共同将一组进程封装成具备独立系统视图和资源边界的执行域，是 Docker、containerd、Kubernetes Pod 隔离模型的内核基础。Namespace 通过内核数据结构为进程维护独立的 PID、网络、挂载、UTS、IPC、USER 等命名空间，使进程看到的系统对象被逻辑分区；Cgroups 通过 cgroupfs 伪文件系统暴露控制器接口，对进程组的 CPU、内存、IO、PIDs 等资源进行配额、优先级、统计和强制回收。它解决的核心问题不是虚拟化，而是多租户进程在同一内核上的可见性隔离与资源公平性。对全栈与 AI 工程师而言，不理解 Namespace/Cgroups，就无法解释容器为何能隔离、为何不能硬隔离、Pod 资源限制如何生效、OOMKill 为何发生、GPU 容器为何仍依赖设备与驱动边界。

### 2. 底层原理剖析
Linux 进程默认共享同一内核视图。Namespace 改变的是进程观察内核对象时使用的命名空间指针，而不是创建新内核。每个进程在 task_struct 中持有 nsproxy，指向其所属的多个 namespace。创建新 namespace 后，新进程在该命名空间内看到的对象集合被重新编号或重映射：PID namespace 使内部进程从 PID 1 开始；mount namespace 使挂载点集合独立；net namespace 使网卡、路由表、iptables、socket 表独立；user namespace 使 UID/GID 映射独立。关键机制是 clone/unshare/setns 系统调用：clone 创建进程时可指定 CLONE_NEWPID、CLONE_NEWNET、CLONE_NEWNS、CLONE_NEWUSER 等标志；unshare 使当前进程脱离原命名空间；setns 使进程加入已有命名空间。

Cgroups 是另一套机制，它不改变视图，只限制资源。内核维护多个控制器，如 cpu、memory、blkio、pids、devices。进程通过写入 cgroupfs 中的 tasks 或 cgroup.procs 被加入控制组。内存控制依赖 mem_cgroup 记录进程页分配并触发阈值检查；CPU 控制依赖 CFS bandwidth control 或 cgroup v2 的 cpu.max；IO 控制依赖块层 throttle 或 io.weight。Cgroups v1 是多棵控制器层级，容易组合复杂且语义不一致；v2 统一为单层级，资源控制器围绕 cgroup 子树协作，是 systemd、Kubernetes 现代运行时的主流方向。

与前端已有概念对比：Namespace 更像 TypeScript 的 module scope 或浏览器 iframe 的 browsing context，决定“你能看到哪些对象、对象如何命名”；Cgroups 更像浏览器对长任务的调度约束、内存上限或 Web Worker 的资源边界，决定“你能消耗多少资源”。但二者不是安全沙箱本体。容器不是虚拟机：它们共享同一内核 syscall 表、内核漏洞面和主机时钟；Namespace 隔离的是资源视图，Cgroups 约束的是资源额度，真正的强安全边界还需要 seccomp、capabilities、LSM/AppArmor/SELinux、只读根文件系统、user namespace 等共同构成。

### 3. 基础代码与实战验证
```text
# 用 unshare 直接验证 namespace 与 cgroup 的基本效果，无需 Docker。

# 1. 创建新的 PID namespace 和 mount namespace 启动 shell。
# 内核为新进程建立独立 nsproxy，PID namespace 中该 shell 成为 PID 1。
sudo unshare --pid --mount --fork --mount-proc /bin/bash

# 在新 shell 中执行：
echo $$
# 输出通常为 1，证明当前进程看到的是新 PID 命名空间，而不是宿主机 PID。

ps -ef
# /proc 被重新挂载为新 mount namespace 下的视图，因此只看到该命名空间内进程。

# 2. 创建 cgroup v2 内存限制。
# 若系统使用 cgroup v2，可执行以下命令；路径可能因发行版略有差异。
sudo mkdir -p /sys/fs/cgroup/demo

# 将内存硬限制设置为 64 MiB。
# 内核将 demo cgroup 的 memory.max 设为阈值，超过后触发 direct reclaim 或 OOM。
echo 67108864 | sudo tee /sys/fs/cgroup/demo/memory.max

# 启动一个受该 cgroup 约束的进程。
# 写入 cgroup.procs 后，shell 及其子进程归属该控制组，页分配计入 mem_cgroup。
echo $BASHPID | sudo tee /sys/fs/cgroup/demo/cgroup.procs

# 在该 shell 中分配超过 64 MiB 的匿名内存，例如运行简单内存填充程序。
# 当 RSS + page cache 计入超过 memory.max，内核优先回收；无法满足时选择进程 OOM kill。

# 3. 观察 cgroup 归属。
cat /proc/self/cgroup
# 输出当前进程所在 cgroup 路径，证明资源控制基于进程组成员关系，而不是文件系统镜像。
```

### 4. 常见误区与进阶思考
误区一：把容器当成虚拟机。Namespace 不提供完整硬件抽象，也不隔离内核本身。容器内进程仍然直接向宿主内核发起 syscall，如果内核存在漏洞或配置不当，容器逃逸风险真实存在。真正的安全模型必须叠加 seccomp 过滤 syscall、capabilities 降权、user namespace 映射、只读 rootfs、MAC 策略。

误区二：认为 Docker/K8s 的 CPU/内存 limit 等同于应用可用资源。Cgroups 限制的是 cgroup 内所有进程的整体资源，不是单个线程或单个 JS/V8 heap。Node.js、Java、Go 运行时如果无法正确读取 cgroup 限额，会按宿主机核数或内存创建线程池、GC 目标，导致超配、抖动或 OOMKill。Kubernetes requests/limits 最终也转化为 cgroup 参数：requests 影响调度和共享权重，limits 对应 cpu.max、memory.max 等硬边界。

进阶思考：在 Kubernetes 中，一个 Pod 内多个容器共享 network namespace，但通常不共享 PID namespace；如果开启 shareProcessNamespace=true，会对故障诊断、信号传播、安全边界产生什么影响？请从 namespace 生命周期、init 进程职责、容器内 PID 1 的信号处理、资源归属和攻击面四个角度分析。
