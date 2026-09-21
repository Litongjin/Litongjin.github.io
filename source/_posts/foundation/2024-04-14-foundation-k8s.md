---
title: "每日基础技术总结 · 2024-04-14 · K8s 调度：亲和性/污点与资源配额"
date: 2024-04-14 20:00:00
categories: [技术分享]
tags: ["技术分享", "云原生与 DevOps（Docker / K8s）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-04-14 · K8s 调度：亲和性/污点与资源配额

## 📚 今日主题

> **K8s 调度：亲和性/污点与资源配额**（云原生与 DevOps（Docker / K8s））

### 1. 核心概念速览
该知识点涵盖 Kubernetes 调度器的核心决策机制，旨在解决Pod在集群中的物理/逻辑位置约束与资源隔离问题。1. 亲和性与反亲和性（Affinity/Anti-Affinity）：通过标签选择器定义Pod部署的偏好或强制约束，分为Node Affinity（节点维度）和Pod Anti-Affinity（Pod间互斥维度），本质是调度器过滤阶段的布尔逻辑判断。2. 污点与容忍（Taints/Tolerations）：节点侧的安全策略，通过给Node打Taint标记其排斥特定工作负载，Pod需声明Toleration才能被调度至该节点，实现类似‘黑名单’的资源预留或隔离。3. 资源配额（Resource Quotas/Limits）：Namespace维度的硬性限制机制，Control Plane Admission Webhook在请求阶段拦截超出CPU、内存或对象数量的API请求，确保多租户环境下的资源公平性与稳定性。工程师必须掌握此机制以应对高可用架构设计、故障域隔离及大规模集群资源治理。

### 2. 底层原理剖析
调度流程解析：
1. 过滤阶段（Filtering）：Scheduler遍历所有候选Node，执行以下布尔校验：
   - Taints匹配：若Node存在Taint且Pod无对应Toleration，直接剔除。
   - Node Affinity：解析MatchExpressions/MatchFields，不满足则剔除。
   - Resource Fit：检查剩余资源是否满足Pod Request，不足则剔除。
2. 打分阶段（Scoring）：对剩余Node进行加权评分，如Balance Spread Score（分散度）、Least Requested Priority（负载均衡）。
3. 绑定阶段（Binding）：SelectBestNode并更新Etcd状态。

与前端的异同对比：
- TS接口 vs K8s Label/Selector：TS接口是编译时契约（Static Contract），K8s Labels是运行时元数据（Runtime Metadata）。Label本身无类型强校验，依赖API Server Admission Logic进行语义验证；TS接口强调类型一致性，K8s Selector强调集合运算（In, Exists, NotIn）。
- Class Inheritance vs Node Topology：前端类的继承是层级关系，K8s Node亲和性是基于拓扑域（TopologyDomain）的平面映射。Node的Zone/AZ属性非继承自Pod，而是通过Label动态关联，调度器需在Etcd中读取Node Status并计算路径距离。
- Memory Limit vs GC：前端GC是自动回收机制，K8s CGroup Limits是内核级硬限制。当容器内存超过Limit，内核OOM Killer介入而非温和回收，这要求开发者必须理解cgroups v1/v2的namespace隔离机制与信号传递逻辑。

### 3. 基础代码与实战验证
```text
# 1. Node Affinity: 强制调度到含有kubernetes.io/os=linux的节点
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution: # 强制规则，不满足则Pending
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/os
            operator: In
            values:
            - linux

# 2. Taints/Tolerations: Node配置Taint
# kubectl taint nodes node1 key=value:NoSchedule
apiVersion: v1
kind: Pod
metadata:
  name: with-toleration
spec:
  tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoSchedule" # 仅容忍NoSchedule，允许PreferNoSchedule

# 3. Resource Quota: Namespace级别的资源上限
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: production
spec:
  hard:
    requests.cpu: "4"       # CPU Core总数上限
    requests.memory: 8Gi    # 内存总量上限
    limits.memory: 16Gi     # 最高限流上限
    pods: "10"              # 最大Pod数量
  selector: {} # 可选字段，针对特定标签的命名空间子集应用
```

### 4. 常见误区与进阶思考
误区一：混淆'Preferred'与'Required'的生命周期意义。requiredDuringSchedulingIgnoredDuringExecution意为调度时必须满足，但若节点Label动态变化导致不再匹配，调度器不会驱逐Pod（IgnoredDuringExecution）；而podAntiAffinity通常也是Required，但需注意如果无法满足，Pod将永远处于Pending状态，导致调度死锁。进阶场景应结合PDB（Pod Disruption Budget）使用。

误区二：误解Resource Limits与Requests的作用域。Requests用于调度时的资源分配预估（Commit），Limits用于运行时的CGroup硬限制（Max）。许多工程师将两者设为相同值，这在突发流量场景下会导致O(OM)风险，而在稀疏分布场景下造成资源碎片化。正确做法是依据基准负载设Requests，依据峰值容忍度设Limits。

思考题：当一个Pod同时配置了Node Affinity (Required) 和 Pod Anti-Affinity (Required)，且集群中只有一个Node满足亲和性条件但该Node上已存在一个具有相同标签的反亲和性Pod，请推演调度器在该Node上的完整决策链与最终状态，并从Etcd交互角度说明如何保证原子性。
