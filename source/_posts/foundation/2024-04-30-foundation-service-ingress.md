---
title: "每日基础技术总结 · 2024-04-30 · Service 与 Ingress：集群内外的流量"
date: 2024-04-30 20:00:00
categories: [技术分享]
tags: ["技术分享", "云原生与 DevOps（Docker / K8s）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-04-30 · Service 与 Ingress：集群内外的流量

## 📚 今日主题

> **Service 与 Ingress：集群内外的流量**（云原生与 DevOps（Docker / K8s））

### 1. 核心概念速览
1. Service：Kubernetes 中的抽象对象，定义了一组 Pod 的逻辑集合及访问策略（L4/L7）。本质是将静态的 Pod IP 动态解耦为稳定的虚拟 IP (VIP) 和 DNS 名称，由 kube-proxy 实现流量转发。解决的是容器 ephemeral（易失）特性导致的后端地址不固定问题。
2. Ingress：集群边缘的应用层（L7）路由控制器。本质是 HTTP/HTTPS 流量在 K8s 集群边界的网关，负责基于 Host、Path 等规则将外部流量分发至内部 Service。其依赖 Nginx/Traefik 等 Ingress Controller 实现实际路由逻辑。解决的是单入口、统一认证、TLS 终结等边界安全与路由需求。
3. 地位：Service 是集群内服务发现与负载均衡的核心机制；Ingress 是集群外向内的标准 API 扩展点。作为工程师必须掌握二者，因为它们是云原生架构中无状态微服务通信契约的基础设施底座。

### 2. 底层原理剖析
1. Service 机制：
- iptables/IPVS 模式：kube-proxy 监听 Endpoints 变化，更新宿主机内核 netfilter 规则（iptables）或 libnlsocket（IPVS），实现 DNAT 将 VIP 流量哈希轮询至 Pod IP。
- eBPF 模式（如 Cilium）：直接在数据包路径挂钩，降低系统调用开销，实现高性能 L4 负载均衡。
- 对比前端 TS Interface：TypeScript Interface 是编译时静态类型检查契约，确保结构一致；Service 声明式 YAML 是运行时动态配置契约，确保流量可达性与负载分布。TS Interface 关注代码静态安全性，Service 关注网络连通性与容错性。

2. Ingress 机制：
- Ingress CRD + Ingress Controller：用户编写 Ingress Resource 描述路由规则 -> ApiServer 持久化 -> Ingress Controller (e.g., Nginx-ingress-controller) 通过 List-Watch 监听到变化 -> 重新渲染配置文件 -> reload Nginx 进程生效。
- L7 vs L4：Service (ClusterIP) 仅处理 TCP/UDP 四层转发；Ingress 解析 HTTP Header、Host、Path，进行七层应用级路由。
- 对比 Java Interface：Java Interface 定义方法签名供类实现；Ingress 定义 URL 路由规则供 Ingress Controller 实现。Java Interface 用于解耦业务逻辑组件；Ingress 用于解耦网络拓扑与业务应用。

### 3. 基础代码与实战验证
```text
# 1. Service 定义示例（YAML）
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  type: ClusterIP        # 默认类型，仅在集群内部可访问，分配内部 VIP
  selector:
    app: my-app          # 标签选择器，自动绑定匹配的 Pod
  ports:
  - port: 80             # Service 层面的端口（VIP 监听端口）
    targetPort: 8080     # 指向 Pod 容器的实际端口
    protocol: TCP        # 传输层协议

# 2. Ingress 定义示例（YAML）
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  ingressClassName: nginx # 指定使用的 Ingress Controller 类
  rules:
  - host: example.com   # L7 路由匹配：基于域名
    http:
      paths:
      - path: /api    # L7 路由匹配：基于路径前缀
        pathType: Prefix
        backend:
          service:
            name: my-app-service # 引用上述 Service
            port:
              number: 80     # 指向 Service 的 port

# 核心逻辑注释：
# 当外部请求到达 Ingress Controller (Nginx)：
# 1. Nginx 读取 Host: example.com 并匹配 pathPrefix: /api
# 2. Nginx 发起内部 TCP 连接，目标为 my-app-service 的 ClusterIP:80
# 3. kube-proxy 收到对 VIP 的连接请求，根据 IPVS/iptables 规则，
#    随机或加权选取一个属于 app=my-app 且端口为 8080 的 Pod IP
# 4. 流量 DNAT 转发至选定 Pod，完成 L4->L7->L4 的完整链路
```

### 4. 常见误区与进阶思考
1. 误区：认为 Ingress 可以替代 Service。
纠正：Ingress 通常后端指向 Service。Ingress 处理七层协议（HTTP/HTTPS），无法直接代理非 HTTP 流量（除非使用 Stream 片段，但复杂度高且非标）。Service 提供稳定的 L4 端点，Ingress 提供灵活的路由能力。二者互补而非替代。

2. 误区：混淆 NodePort 与 LoadBalancer。
纠正：NodePort 在每个节点开放高位端口，流量经节点 IP 进入 kube-proxy 再到 Pod；LoadBalancer 依赖云厂商 LB 硬件，流量经云 LB 健康检查后直达 Pod 或经过 Node。生产环境极少直接用 NodePort 暴露服务给公网，因其缺乏高级 L7 特性（如 SSL 卸载、限流）。

思考题：如果两个不同的 Ingress 资源定义了相同 Host 和 Path 但指向不同的 Service，Kubernetes 如何处理冲突？请结合 Ingress Controller 的实现原理（如 Nginx config generation）分析其优先级策略或失败结果。
