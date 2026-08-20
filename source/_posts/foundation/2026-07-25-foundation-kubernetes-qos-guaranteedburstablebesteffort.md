---
title: "每日基础技术总结 · 2026-07-25 · Kubernetes 的 QoS 分类与驱逐顺序：Guaranteed/Burstable/BestEffort"
date: 2026-07-25 08:00:00
categories: [技术分享]
tags: ["技术分享", "架构与设计"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-25 · Kubernetes 的 QoS 分类与驱逐顺序：Guaranteed/Burstable/BestEffort

## 📚 今日主题

> **Kubernetes 的 QoS 分类与驱逐顺序：Guaranteed/Burstable/BestEffort**（架构与设计）

### 1. 核心概念速览
Kubernetes QoS（Quality of Service）分类是节点资源管理的基础机制，它将Pod划分为Guaranteed、Burstable、BestEffort三个等级，依据每个容器的requests与limits的声明关系。其本质是：在资源竞争（如内存压力、磁盘压力、PID耗尽）时，定义不同Pod的可牺牲顺序，保证系统整体稳定性。机制上，kubelet根据Pod中所有容器的资源声明计算QoS类，并在节点压力驱逐时按“BestEffort → Burstable → Guaranteed”的优先级执行，同时在cgroup和OOM评分中体现差异。该机制位于Kubernetes调度与节点生命周期管理的交叉点，直接决定工作负载的可靠性与资源利用率。专业工程师必须掌握，因为它影响Pod的存活概率、集群容量规划、以及故障场景下的行为预期，是理解Kubernetes资源治理和稳定性设计的关键。

### 2. 底层原理剖析
底层机制分为三个层次：1）QoS分类规则：kubelet在Pod创建时计算。若Pod内每个容器（包括init容器）都设置了requests和limits，且对应资源（CPU/内存）的requests等于limits，则为Guaranteed；若至少一个容器设置了requests或limits（但不同时满足所有相等条件），则为Burstable；若所有容器均未设置requests和limits，则为BestEffort。2）驱逐顺序：kubelet通过Eviction Manager监控节点资源压力（如memory.available、nodefs.available、imagefs.available、pid.available等）。当超过阈值，进入压力状态，kubelet按照QoS类从低到高驱逐Pod，以释放资源。具体顺序：先驱逐BestEffort，再Burstable，最后Guaranteed。但实际驱逐会结合Pod优先级（PriorityClass）和资源用量，同一QoS内先驱逐超出requests更多的Pod。3）OOM评分与cgroup：kubelet为每个容器设置OOM Score，范围[-1000, 1000]。Guaranteed的容器默认-998，Burstable按内存使用量与requests的偏差动态计算，BestEffort通常为1000，使得内核在OOM时优先杀死得分高的进程。同时，cgroup的oom_control和memory limit确保资源隔离。与前端概念的对比：Java接口是编译期类型约束，声明行为契约；TS接口是结构类型，编译后消失。Kubernetes QoS则是一种运行时资源保障契约，声明在Pod Spec中，但由kubelet在节点上强制执行，并影响内核OOM行为。类似地，Java接口和TS接口都定义“形状”，但Java在运行时保留类型信息（通过反射），而TS在编译时擦除类型；K8s的QoS也类似——requests/limits在API层是声明，但运行时的cgroup和OOM评分才真正“执行”了这个声明。

### 3. 基础代码与实战验证
```text
# 验证QoS分类的极简YAML示例

# 1. Guaranteed: 所有容器均设置requests=limits
apiVersion: v1
kind: Pod
metadata:
  name: qos-guaranteed
spec:
  containers:
  - name: c1
    image: busybox
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "128Mi"
        cpu: "100m"

# 2. Burstable: 至少一个容器设置了requests或limits，但存在不等于或未设置
apiVersion: v1
kind: Pod
metadata:
  name: qos-burstable
spec:
  containers:
  - name: c1
    image: busybox
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"  # 不等于requests
        cpu: "200m"

# 3. BestEffort: 所有容器均未设置requests和limits
apiVersion: v1
kind: Pod
metadata:
  name: qos-besteffort
spec:
  containers:
  - name: c1
    image: busybox

# 验证命令：
# kubectl apply -f <file>
# kubectl get pod -o jsonpath='{.status.qosClass}' <pod-name>
# 输出分别为 Guaranteed, Burstable, BestEffort
# 观察驱逐行为：在节点上人为触发内存压力，查看kubelet日志或pod事件，验证驱逐顺序。
```

### 4. 常见误区与进阶思考
误区1：认为Guaranteed Pod绝对不会被驱逐。实际虽然Guaranteed优先级最高，但当节点出现硬件故障、系统组件异常或kubelet需要保护节点本身时，Guaranteed Pod也可能被驱逐。另外，若Guaranteed Pod超过其limit，内核OOM仍可能杀死其容器（OOM Score为-998但非绝对安全）。误区2：混淆QoS类与PriorityClass。QoS类由资源声明自动推导，反映资源可牺牲性；PriorityClass是用户自定义的调度优先级，影响调度和驱逐（当存在多级优先级时，高优先级Pod先于低优先级被驱逐？实际上驱逐顺序先看QoS，再看Priority）。思考题：一个Pod包含两个容器：容器A设置了CPU的requests和limits相等（均为100m），但未设置内存；容器B设置了内存的requests和limits相等（均为128Mi），但未设置CPU。请问该Pod的QoS分类是什么？并说明原因（提示：QoS分类要求每个容器对所有资源（CPU和内存）都满足相应的条件）。
