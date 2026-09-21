---
title: "每日基础技术总结 · 2024-07-12 · 容器网络：bridge/host/overlay 模型"
date: 2024-07-12 20:00:00
categories: [技术分享]
tags: ["技术分享", "云原生与 DevOps（Docker / K8s）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-07-12 · 容器网络：bridge/host/overlay 模型

## 📚 今日主题

> **容器网络：bridge/host/overlay 模型**（云原生与 DevOps（Docker / K8s））

### 1. 核心概念速览
容器网络模型旨在解决容器进程隔离后的命名空间冲突与通信问题，核心机制是 Linux 网络命名空间（Network Namespace）与虚拟以太网对（veth pair）的绑定。
1. Bridge 模式：默认驱动。通过网桥（Linux bridge, 如 docker0）连接宿主机的物理网卡与各容器的 veth pair，实现容器间二层交换及 NAT 出站访问。本质是在内核中添加一个虚拟交换机，每个容器拥有独立 IP，宿主通过 iptables/ebtables 进行 SNAT/DNAT 转换。
2. Host 模式：容器共享宿主机的网络命名空间，无独立 IP、端口映射或隔离。直接调用宿主机 TCP/IP 栈，性能最高，但无法实现多容器端口复用，适用于无需隔离的网络场景或底层监控代理。
3. Overlay 模式：用于跨主机集群（如 Docker Swarm/K8s CNI）。基于 VXLAN 隧道技术，将原始以太网帧封装在 UDP 数据包中，通过宿主机物理网络传输，解耦逻辑 L2 网络与物理 L3 网络，实现大规模分布式网络的平面化。
专业工程师必须掌握它，因为云原生应用的网络拓扑、调试（tcpdump/traceroute）、安全策略（NetworkPolicy）均建立在此基础架构之上。

### 2. 底层原理剖析
底层运行机制解析：

【Bridge 模型】
- 初始化：Daemon 创建 Linux bridge (br0) 并分配网段 (e.g., 172.17.42.1/16)。
- 容器启动：创建一对 veth pair (vethA, vethB)。一端置于容器网络命名空间 ns_Container 作为 eth0；另一端挂载至 br0 成为端口。
- 路由/NAT：利用 iptables masquerade 规则，将源自容器网段的数据包源地址替换为宿主机出口 IP，实现出站互联网访问。容器间通信通过 br0 二层转发，无需经过内核路由层，延迟极低。

【Host 模型】
- 机制：运行时不创建新的 network namespace，而是复用 init namespace。容器内进程直接绑定宿主机端口，IP 即宿主机 IP。
- 对比前端接口概念：类似 TS 中的 `any` 类型或 Java 中的反射直调——绕过了类型系统/抽象层的约束，直接操作底层资源。优点是零拷贝开销，缺点是破坏了封装性（Encapsulation），导致依赖隐式耦合。

【Overlay 模型】
- 封装协议：VXLAN (RFC 7348)。
- 数据流：Container A -> veth -> Container B's NetNS -> Overlay Driver -> Local VTEP (Virtual Tunnel End Point) 封装 VXLAN Header + Outer UDP/ETH/IP -> Physical Network -> Remote VTEP -> De-virtualize -> Container B.
- 关键点：Overlay 头增加了约 50-60 Bytes 开销，需调整 MTU（通常设为 1450 而非 1500）以防爆片重组开销。

对比前端知识体系：
- Bridge 类似于 Webpack/Vite 的本地开发服务器（localhost proxy），对外提供统一入口，内部隔离模块。
- Host 类似于 Node.js 进程中全局事件循环的直接监听，无上下文切换损耗但易产生命名空间污染。
- Overlay 类似于 CDN 边缘节点的回源机制，将逻辑链路映射到物理骨干网。

### 3. 基础代码与实战验证
```text
// 使用 ip netns 命令直观演示 Linux 网络命名空间与 veth pair 原理
// 需 root 权限执行

# 1. 创建虚拟以太网对 (veth pair)，类比创建两个相互连接的插座
ip link add veth-a type veth peer name veth-b

# 2. 创建独立网络命名空间 (Namespace A)，类比创建一个全新的浏览器沙箱环境
ip netns add netns-a

# 3. 将 veth-b 移入命名空间 A，并将该接口启用
ip link set veth-b netns netns-a
ip netns exec netns-a ip link set dev veth-b up
ip netns exec netns-a ip addr add 10.0.0.2/24 dev veth-b

# 4. 配置主机端 veth-a 并入网桥 (模拟 bridge 模式)
# 此处省略 brctl 或 bridge link 创建网桥步骤，假设已创建 bridge0
bridge link add dev veth-a master bridge0
ip link set dev veth-a up
ip addr add 10.0.0.1/24 dev veth-a

# 5. 验证连通性 (Ping 测试)
ip netns exec netns-a ping -c 1 10.0.0.1

# 底层原理注释：
# - 'ip link add ... type veth': 在内核中申请两块虚拟网卡内存，强制关联 MAC 地址，任何一侧收包即时触发另一侧发包。
# - 'ip netns': 修改进程所属的 networking namespace 指针表，使后续 syscall (socket/bind/listen) 仅在当前命名空间的视图下生效。
# - 未使用 host 模式的 isolate，因此 veth-a 仍可见于宿主机全局命名空间。
```

### 4. 常见误区与进阶思考
1. **MTU 陷阱**：启用 Overlay 网络时，若宿主机物理网卡的 MTU 仍为默认的 1500，而 VXLAN 头部增加了额外负载（~50 Bytes），会导致数据包分片甚至丢弃。正确做法是将宿主机出站网卡 MTU 调整为 1450 或更低（如 1400），并确保容器内镜像的 ifcfg/iptables 匹配此值。这是生产环境丢包的最常见原因。

2. **Host 模式的安全误区**：认为 Host 模式能显著提升性能是事实，但误以为可以随意暴露端口。由于容器共享宿主机网络栈，若在容器内 bind 0.0.0.0:80，实际上暴露的是宿主机所有网卡 IP。这与 K8s DaemonSet 的 hostPort 特性类似，极易造成端口冲突和安全边界穿透。对于高并发微服务，应优先采用 eBPF 加速或 DPDK 方案优化标准 Bridge/Host 网络的性能瓶颈，而非盲目退守 Host 模式破坏隔离性。

思考题：
在 Kubernetes 环境中，若 Pod A (NetNS 1) 通过 CoreDNS 解析 Pod B 的 ClusterIP 发起 HTTP 请求，数据包从 Pod A 发出到到达 Pod B 的进程缓冲区，经历了哪些内核态组件的跳转？如果中间经过了一个 Service LoadBalancer 类型的节点，kube-proxy 的 iptables/ipvs 规则在其中扮演了什么具体的包过滤/转发角色？
