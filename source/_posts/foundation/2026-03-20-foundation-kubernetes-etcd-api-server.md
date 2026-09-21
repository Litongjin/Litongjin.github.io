---
title: "每日基础技术总结 · 2026-03-20 · Kubernetes 架构：Etcd/API Server/调度器"
date: 2026-03-20 20:00:00
categories: [技术分享]
tags: ["技术分享", "云原生与 DevOps（Docker / K8s）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-03-20 · Kubernetes 架构：Etcd/API Server/调度器

## 📚 今日主题

> **Kubernetes 架构：Etcd/API Server/调度器**（云原生与 DevOps（Docker / K8s））

### 1. 核心概念速览
1. 核心概念速览
Kubernetes (K8s) 是容器编排引擎，其控制平面由 Etcd、API Server 和 Scheduler 构成，本质是一个分布式一致性状态机系统。
- Etcd: 强一致性的分布式键值存储（基于 Raft 协议）。它解决‘声明式 API 的状态持久化与广播’问题，是所有资源的唯一事实来源（Single Source of Truth）。
- API Server: 控制平面的前端入口，负责认证、授权、准入控制及资源 CRUD。它将用户提交的 JSON/YAML 序列化为内部对象，写入 Etcd。它不执行业务逻辑，仅作为请求路由器和校验器。
- Scheduler: 决策模块。监听未绑定 Node 的 PodSpec，通过 ‘预选（Fit）’ 和 ‘优选（Score）’ 算法将 Pod 调度至最优 Node，并在 Etcd 中更新 Node 上的 podName 字段以绑定 Pod。
专业工程师必须掌握：因为云原生应用部署不再是静态文件拷贝，而是动态状态同步过程；理解此架构才能调试集群状态不一致、调度失败及持久化数据丢失等深层问题。

### 2. 底层原理剖析
2. 底层原理剖析
运行机制遵循 '期望状态 -> 实际状态' 的闭环反馈机制（Reconciliation Loop）。
- 流程：User/Ctrl Plane -> API Server (Auth/Validate) -> Etcd (Persist) -> Controller Manager (Watch ETCD, Compare Expect vs Reality) -> kubelet/pod on Nodes.
- Watch 机制：所有组件通过 HTTP Long-Polling (SSE) 向 API Server 注册 Watch 事件流。当 Etcd 数据变更时，API Server 推送 delta 给各组件，触发二次协调。这是 K8s 实现最终一致性和热加载的核心。
- 对比前端：
  * 前端 React/Vue 是单向数据流或双向绑定在 DOM 层的抽象，K8s 是跨进程、跨主机的分布式状态同步。
  * Java Interface 定义契约，TypeScript Interface 编译期类型检查；而 K8s Resource Definition (CRD/API) 既是运行时序列化协议，又是分布式事务的原子操作单元。API Server 类似 GraphQL 的统一查询层，但增加了严格的 RBAC 和 Admission Webhook 拦截链。

### 3. 基础代码与实战验证
```text
3. 基础代码与实战验证
使用 kubectl CLI 模拟底层交互逻辑，展示 API Server 与 Etcd 的同步过程。

# 步骤 1: 创建 Deployment (POST /apis/apps/v1/namespaces/default/deployments)
kubectl create deployment nginx --image=nginx -o yaml > deploy.yaml

# 步骤 2: 查看 Etcd 中的原始存储结构 (模拟 kubectl get etcdkey... 需直连 etcdctl)
# 本质：JSON 被序列化存入 /registry/deployments/default/nginx
etcdctl get /registry/deployments/default/nginx --print-value-only

# 步骤 3: 触发调度循环 (Scheduler Watch 到新的 unscheduled Pod)
kubectl get pod | grep nginx
# 输出示例: NAME READY STATUS RESTARTS AGE
# nginx-xxxxx 0/1 Pending

# 步骤 4: 验证调度结果 (Scheduler 更新了 Node 对象的 PodRef，Controller Manager 启动容器)
kubectl describe pod <pod-name>
# 关注 Events: Scheduled -> Started -> Created -> Started container

# 伪代码逻辑解释:
/* 
function reconcileLoop() {
  currentDesiredState = API_Server.GetResourceFromEtcd(type, name);
  currentState = NodeAgent.ReportStatus();
  diff = currentDesiredState - currentState;
  if (diff != null) {
    SchedulePod(diff.PodSpec); // 调用 Scheduler 算法
    CreateContainerOnNode(diff.PodSpec.TargetNode); // 调用 CRI (containerd/cri-o)
  }
}
*/
```

### 4. 常见误区与进阶思考
4. 常见误区与进阶思考
- 误区 1: 认为 Scheduler 直接创建 Pod。实际上，Scheduler 只负责‘分配’（Binding），真正创建容器的是目标 Node 上的 kubelet（通过 CRI 接口调用 containerd/docker）。若不理解此分离，无法定位‘调度成功但容器未起’的问题。
- 误区 2: 混淆 API Server 与业务服务。API Server 是无状态的，但它是同步阻塞的权威节点；业务微服务通常无状态且异步。错误地依赖 API Server 处理高频业务读写会导致集群雪崩。
- 深度思考题：
  在大规模集群中，如果 Etcd 出现网络分区导致 leader 切换，此时正在进行的 `kubectl apply` 请求会发生什么？API Server 的 Write Request 是如何利用 Revision 号和 Quorum 保证不丢失数据且不产生脏读的？请结合 Raft Commit 机制描述从 Client 到 Leader 再到 Follower 的数据流转路径。
