---
title: "每日基础技术总结 · 2026-09-30 · 存储编排：PV/PVC/StorageClass"
date: 2026-09-30 07:05:48
categories: [技术分享]
tags: ["技术分享", "云原生与 DevOps（Docker / K8s）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-30 · 存储编排：PV/PVC/StorageClass

## 📚 今日主题

> **存储编排：PV/PVC/StorageClass**（云原生与 DevOps（Docker / K8s））

### 1. 核心概念速览
PV（PersistentVolume）是集群中由管理员或动态供给创建的存储资源抽象，本质上是将外部存储系统（NFS、Ceph、云盘等）的挂载能力封装为集群可调度的 API 对象；PVC（PersistentVolumeClaim）是用户对存储资源的请求声明，本质上是资源需求与访问模式的约束集合；StorageClass 则是动态供给的模板，定义了存储类型、回收策略、挂载参数与供给者（Provisioner）。它们共同构成了 Kubernetes 存储编排的『控制面与数据面解耦』机制：用户只声明需求（PVC），系统负责将需求匹配到满足条件的 PV，或通过 StorageClass 动态创建 PV，并将实际存储挂载到 Pod 的文件系统路径。该机制解决了静态配置存储与动态调度之间不可调和的问题——存储生命周期与 Pod 生命周期无关，且存储的供给、绑定、使用、回收全部由控制平面自动管理。在整个计算机体系里，这属于资源抽象与编排层，类比操作系统的虚拟内存与物理内存的映射关系，但粒度在分布式存储之上。专业工程师必须掌握，因为它是构建有状态服务（数据库、消息队列）的基石，不理解 PV/PVC 就无法在 Kubernetes 上正确设计持久化架构，更无法诊断存储相关的故障。

### 2. 底层原理剖析
PV/PVC 的底层运行机制可以拆解为三个阶段：供给、绑定与消费。供给分为静态和动态：静态供给时，管理员预先创建 PV 对象，其中包含存储类型（如 NFS 的 server/path、CSI 的 volumeHandle）、容量、访问模式（ReadWriteOnce/ReadOnlyMany/ReadWriteMany）以及回收策略（Retain/Delete/Recycle）；动态供给时，当 PVC 无法匹配任何现存 PV，就会触发 StorageClass 对应的 Provisioner（通常是一个 CSI 插件或内置插件）创建底层存储并生成 PV。绑定过程采用双向匹配算法：控制平面（kube-controller-manager 中的 PersistentVolumeController）监听 PVC 和 PV 的创建事件，遍历所有 PV，找出满足容量（>=）、访问模式子集、storageClassName 一致且未被绑定的 PV，将 PV 的 claimRef 指向该 PVC，完成绑定。绑定本质上是写入 API Server 的 etcd，并触发相关 Finalizer 与回收策略的注册。消费过程由 kubelet 负责：当 Pod 调度到节点后，kubelet 根据 Pod 声明的 volume 数组，找到对应的 PVC 和绑定的 PV，调用具体的 volume plugin（如 CSI、NFS、hostPath）进行挂载操作——先做阶段挂载（全局挂载到节点），再做全局挂载到 Pod 的 volume 目录，最后 bind-mount 到容器路径。整个过程中，PVC 与 PV 的访问模式必须严格匹配，否则无法绑定；容量是绝对值，PV 大于等于 PVC 即可绑定，但配额取 PVC 的请求值。与前端概念对比：PVC 相当于 TypeScript 的 interface——它只描述『我需要什么』（容量、模式），而 PV 类似于 interface 的实现类，StorageClass 则接近于依赖注入的工厂函数——根据接口描述动态创建实现。更精确地说，PVC 是『存储接口』，PV 是『具体存储实例』，StorageClass 是『存储工厂』。但与外界的 Java 接口不同，K8s 的绑定是隐式的，由控制器根据语义匹配，不要求显式引用；且 PV 与 PVC 之间存在生命周期绑定关系，类似 TypeScript 接口与实现类的动态注入，但具有状态与回收语义。

### 3. 基础代码与实战验证
以下为验证 PV/PVC/StorageClass 绑定与动态供给机制的极简 YAML 序列，不依赖复杂框架。

1. 定义 StorageClass（动态供给入口）：
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/no-provisioner  # 这里使用 no-provisioner 只做演示，实际 CSI 需要接存储后端
reclaimPolicy: Delete  # 当 PVC 删除时，PV 及其底层存储被删除
volumeBindingMode: Immediate  # 立即绑定，而非 WaitForFirstConsumer
```

2. 创建 PVC（用户声明需求）：
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce  # 单节点读写，对应底层存储的访问限制
  resources:
    requests:
      storage: 1Gi  # 请求容量，控制平面会寻找 PV 满足此大小
  storageClassName: fast  # 指定 StorageClass，若 none 则禁用动态供给
```

3. 观察绑定过程（验证控制平面自动创建 PV）：
执行 `kubectl get pv, pvc`，当 PVC 提交后，若 StorageClass 有 provisioner 逻辑，则控制平面会创建 PV 并绑定；若使用 no-provisioner，则 PVC 会处于 Pending，直到人工创建匹配的 PV

4. 手动创建静态 PV（用于演示静态供给匹配）：
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 2Gi  # 容量大于 PVC 请求的 1Gi，允许绑定
  accessModes:
    - ReadWriteOnce  # 访问模式需包含 PVC 的模式
  storageClassName: fast  # 必须与 PVC 的 storageClassName 一致
  hostPath:
    path: /tmp/demo-storage  # 实际存储路径，仅演示用，生产应使用 CSI
```

关键机制解析：
- PVC 创建后，PersistentVolumeController 会同时扫描集群中的 PV 和 StorageClass，执行 capacity 比较与 accessModes 子集校验。
- 匹配成功后，控制器将 PV 的 `status.phase` 从 Available 变为 Bound，并在 PV 中写入 `claimRef` 指向 PVC 的命名空间与名称。
- 若 PV 被手动创建且未指定 storageClassName，且 PVC 也未指定，则系统会尝试绑定；但如果集群启用了默认 StorageClass，则双方必须显式一致。

以下为验证挂载到 Pod 的片段：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: data
          mountPath: /usr/share/nginx/html  # 容器内路径
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: my-pvc  # 引用 PVC，kubelet 会解析其对应的 PV
```
kubelet 在 Pod 启动前会调用 volume plugin 将 PV 的存储挂载到节点上，然后 bind-mount 到容器指定路径，过程与 `kubectl exec` 的 namespace 隔离无关，而是通过 mount 系统调用实现。

### 4. 常见误区与进阶思考
误区一：认为 PV 的容量就是实际可用容量。实际中 PV 是一个资源抽象，容量只是声明值，底层存储可能超卖或存在文件系统开销，若使用 hostPath 或某些网络存储，容量限制根本不会强制，读写大小超出声明时可能不报错直到磁盘写满。误区二：认为 PVC 与 PV 的绑定是持久化的物理连接，忽略回收策略。如果 PV 的 reclaimPolicy 是 Retain，删除 PVC 后 PV 仍会存在但其 phase 变为 Released，且旧的 claimRef 仍指向已删除的 PVC，导致该 PV 不会被重新绑定，必须人工清理保留的数据和 claimRef 才能复用。很多工程师因此删了 PVC 后，PV 无法自动挂到新 PVC，误以为系统故障。

思考题：在 volumeBindingMode: WaitForFirstConsumer 模式下，PVC 不会被立即绑定，而是等到某个 Pod 引用该 PVC 并被调度时，调度器才根据 Pod 的节点选择以及 PV 的拓扑约束（如 zone 可用性）来动态绑定。请思考这种延迟绑定是如何影响 StorageClass 中 provisioner 创建 PV 时的参数选择？如果调度器与控制平面之间没有同步机制，会发生什么竞态？
