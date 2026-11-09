---
title: "每日基础技术总结 · 2026-11-10 · 负载均衡器源IP保留（Source IP Preservation）通过Direct Server Return或隧道实现"
date: 2026-11-10 07:03:20
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-10 · 负载均衡器源IP保留（Source IP Preservation）通过Direct Server Return或隧道实现

## 📚 今日主题

> **负载均衡器源IP保留（Source IP Preservation）通过Direct Server Return或隧道实现**（网络基础）

### 1. 核心概念速览
1. 核心概念速览

源 IP 保留（Source IP Preservation）指负载均衡器把请求转发到后端服务器时，后端传输层看到的对端地址仍是真实客户端 IP，而不是负载均衡器地址。其本质是：请求路径不执行源地址 SNAT，且响应路径允许后端直接回客户端，不强制经过负载均衡器。

它解决的是 Full NAT/SNAT 负载均衡导致后端 peer 地址退化为 LB 地址的问题。后端应用若拿不到真实客户端 IP，会破坏 ACL、审计、限流、地域识别、会话保持、计费、风控、TLS 透传以及非 HTTP 协议的真实对端识别。应用层 X-Forwarded-For 只能覆盖 HTTP，且可伪造；源 IP 保留发生在网络层/传输层五元组层面，对 TCP/UDP 透明。

两种典型实现：

- DSR（Direct Server Return，直接路由）：LB 与后端在同一二层广播域。LB 转发时只改写以太网目的 MAC 为后端网卡 MAC，IP 头中的源 IP=客户端、目的 IP=VIP 保持不变。后端把 VIP 配置在 lo 或非 ARP 响应接口上，回包以 VIP 为源直接发给客户端，不经过 LB。
- 隧道：LB 用 IPIP/GRE 等把原始 IP 包封装为外层 LB->RS 的包，内层保留客户端->VIP 五元组。后端解封装后看到真实客户端 IP，回包同样直出。隧道可跨子网，但引入封装开销与 MTU 问题。

该机制位于 L4 负载均衡与网络转发层，是 IPVS DR/Tunnel、云负载均衡透明模式、Kubernetes 外部流量策略与后端真实 IP 治理的基础。专业工程师必须掌握，因为它决定后端看到的 socket 对端、回程路由、防火墙非对称路径、连接跟踪与故障摘除方式。

### 2. 底层原理剖析
2. 底层原理剖析

请求方向：客户端 C 发送 IP(CIP:CPORT -> VIP:VPORT)。LB 选后端 RS 后：

- DSR：LB 不改 IP 头，只把二层帧目的 MAC 改为 RS_MAC，源 MAC 为 LB_MAC，从同网段接口发出。RS 内核因 VIP 在本地（常见 lo/32）而接受该包，应用 accept 得到 peer=CIP。RS 回包为 IP(VIP:VPORT -> CIP:CPORT)，按路由表直接发往客户端或默认网关，不经过 LB。
- 隧道：LB 构造外层 IP(LB_IP -> RS_IP)，载荷是完整内层 IP(CIP -> VIP)。RS 解封装后把内层包交付本地 VIP。回包仍为 IP(VIP -> CIP)，直出。

关键约束：

- RS 必须能本地接收目的 VIP 的包，通常把 VIP 配在 lo；同时抑制 VIP 的 ARP 应答（arp_ignore=1、arp_announce=2），避免与 LB 的 VIP ARP 冲突。
- 回程路由必须直出到客户端侧，不能默认指回 LB，否则形成三角路由并可能被 rp_filter/防火墙丢弃。
- LB 只看到请求方向流量，通常看不到 SYN-ACK 与数据回包，因此不能依赖双向连接跟踪做响应侧 NAT/L7 处理。
- 隧道封装使有效 MTU 减少（IPIP 减 20 字节，GRE 更多），需 PMTUD 或 TCP MSS clamp。

伪代码流程：

on client: ARP(VIP) -> LB_MAC; send frame(dst=LB_MAC, IP(CIP->VIP))
LB: if dst_ip==VIP: rs=select_backend();
    DSR: frame.dst=RS_MAC; frame.src=LB_MAC; ip_header.unchanged; send
    Tunnel: outer=IP(LB_IP->RS_IP); payload=IP(CIP->VIP); send
RS: if dst_ip==VIP and VIP is local: accept; app.peer=CIP;
    reply=IP(VIP->CIP); route(reply) direct_to_client; send

与前端已有知识对比：Nginx 反向代理在应用层终止并重建连接，后端 accept 到的 peer 是代理地址，真实 IP 只能通过 X-Forwarded-For、Forwarded 或 PROXY protocol 带外传递。XFF 是 HTTP 头，属于业务层附加元数据，可被客户端伪造，必须在可信边界清洗；PROXY protocol 是 TCP 层前置头，需要 LB 与后端协议支持，但不改变 IP 包。DSR/隧道保留的是原始网络层五元组，后端 socket 对端直接是 CIP，等价于保持原始 socket 地址，而不是在应用头里塞字段。

### 3. 基础代码与实战验证
```text
3. 基础代码与实战验证

以下用 Linux netns 验证 DSR 核心路径。假设已建好 client/lb/rs 三个 netns 并挂到同一 bridge：client=10.0.0.2，lb=10.0.0.10，rs=10.0.0.20，VIP=10.0.0.100；veth 名分别为 veth_client/veth_lb/veth_rs。

# RS：VIP 只放 lo，避免参与二层 ARP；抑制 VIP 的 ARP 通告/应答
ip netns exec rs sysctl -w net.ipv4.conf.all.arp_ignore=1
ip netns exec rs sysctl -w net.ipv4.conf.all.arp_announce=2
ip netns exec rs sysctl -w net.ipv4.conf.all.rp_filter=0
ip netns exec rs ip addr add 10.0.0.100/32 dev lo

# LB：开启转发；把 VIP 路由到 RS。转发时目的 IP 仍为 VIP，目的 MAC 改为 RS_MAC
ip netns exec lb sysctl -w net.ipv4.ip_forward=1
ip netns exec lb ip route add 10.0.0.100/32 via 10.0.0.20 dev veth_lb

# client：静态 ARP 把 VIP 解析为 LB_MAC，使请求帧先到 LB。生产环境由 LB 持有 VIP 并响应 ARP
ip netns exec client ip neigh replace 10.0.0.100 lladdr LB_MAC dev veth_client

# RS 上启动监听 VIP:8080 的服务（nc 参数随实现不同，核心是 bind VIP）
ip netns exec rs nc -l -s 10.0.0.100 -p 8080

# 验证：RS 的 tcpdump 应看到 10.0.0.2 -> 10.0.0.100，而不是 10.0.0.10 -> 10.0.0.100
ip netns exec rs tcpdump -ni veth_rs 'tcp port 8080'
ip netns exec client nc 10.0.0.100 8080

# 隧道对照：LB 上建 IPIP，把 VIP 路由进 tun0；RS 建同名 tun0 并 up 以解封装
# ip netns exec lb ip tunnel add tun0 mode ipip local 10.0.0.10 remote 10.0.0.20
# ip netns exec lb ip link set tun0 up
# ip netns exec lb ip route replace 10.0.0.100/32 dev tun0
# 外层 IP(LB->RS) 只负责传输，内层 IP(CIP->VIP) 保留源 IP；RS 回包仍以 VIP 为源直出。

关键点：RS 上 tcpdump 看到 CIP 即证明请求路径未做 SNAT；客户端抓包看到响应源为 VIP 且不经过 LB，即证明回程直出。
```

### 4. 常见误区与进阶思考
4. 常见误区与进阶思考

误区一：把 X-Forwarded-For 或 PROXY protocol 当成源 IP 保留。XFF 是应用层 HTTP 头，可伪造且只适用于 HTTP；PROXY protocol 是 TCP 前置头，需要两端支持。它们传递的是带外真实 IP 信息，不改变后端 socket 的对端地址。DSR/隧道保留的是网络层五元组，后端 accept 直接得到 CIP。

误区二：认为 DSR 下回包也要经过 LB，或 LB 能像 Full NAT 一样看到完整双向连接。DSR 的回包由 RS 直接发往客户端，LB 通常只观测请求方向；因此响应侧 L7 处理、双向 conntrack、响应侧 NAT 都不成立。若回程路由误指 LB，会形成非对称路径并被防火墙/rp_filter 丢弃；若 RS 未抑制 VIP ARP，会造成 VIP ARP 冲突和流量漂移。

进阶思考：DSR 下 LB 看不到 SYN-ACK 和数据回包，只能基于请求方向维护连接表。若后端进程崩溃但没有发出 RST，或后端需要优雅摘除，LB 如何可靠判断连接状态并避免把新连接调度到失效后端？请从 TCP 状态机单向观测、连接表超时、主动健康检查与连接 draining 的关系解释。
