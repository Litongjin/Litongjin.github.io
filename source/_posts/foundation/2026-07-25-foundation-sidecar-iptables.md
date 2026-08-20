---
title: "每日基础技术总结 · 2026-07-25 · 服务网格中 sidecar 的 iptables 流量劫持与启动顺序"
date: 2026-07-25 08:00:00
categories: [技术分享]
tags: ["技术分享", "架构与设计"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-25 · 服务网格中 sidecar 的 iptables 流量劫持与启动顺序

## 📚 今日主题

> **服务网格中 sidecar 的 iptables 流量劫持与启动顺序**（架构与设计）

### 1. 核心概念速览
服务网格中的 sidecar 流量劫持，本质是通过 Linux 内核 netfilter 框架的 iptables 规则，在数据包进入用户态应用之前或离开应用之后，强制将流量重定向到 sidecar 代理（如 Envoy）的监听端口。其核心机制是修改数据包的目标地址（DNAT）或使用透明代理（TPROXY），使应用进程无感知地通过 sidecar 转发流量。该机制解决的是『如何在不修改应用代码的情况下，将服务间通信纳入网格管控』的问题，是服务网格数据面实现流量治理、可观测性、安全策略的基础设施。启动顺序问题则源于 iptables 规则必须在 sidecar 进程启动之前就绪，否则应用启动时发出的流量无法被劫持，导致流量绕过网格。它在整个系统体系中的位置：位于容器网络命名空间与内核协议栈之间，是云原生基础设施中网络数据面的关键一环。专业工程师必须掌握它，因为流量劫持是服务网格的根基，理解它才能诊断流量异常、设计安全策略、优化启动性能，并避免在 Kubernetes 环境中因 init 容器或 sidecar 注入顺序导致的故障。

### 2. 底层原理剖析
iptables 劫持的底层原理：Linux 数据包路径上，netfilter 在协议栈中设置了多个钩子点（PREROUTING、INPUT、FORWARD、OUTPUT、POSTROUTING）。对于进出 Pod 的流量，关键路径如下：
1. 入站流量（外部→Pod）：数据包到达主机网卡，进入 PREROUTING 链，匹配规则（如 '-p tcp --dport 8080 -j REDIRECT --to-port 15006'），将目标端口改写为 sidecar 的监听端口，然后经 INPUT 链进入本机，sidecar 进程接收该连接。
2. 出站流量（Pod→外部）：应用进程发起连接，数据包在本机生成，经过 OUTPUT 链。匹配规则（如 '-p tcp --dport 80 -j REDIRECT --to-port 15001'），将目标地址/端口改为 sidecar 的监听端口，然后经过 POSTROUTING 发出，但实际被 sidecar 接收。
关键点：REDIRECT 规则仅作用于本机生成的流量，且改写的目标地址是 loopback 接口，因此 sidecar 必须监听在对应端口。TPROXY 则允许在不修改目标地址的情况下，将流量转发到本地 socket，保留原始目标地址供代理进行路由决策。
启动顺序的底层逻辑：iptables 规则存储在共享的内核表（netfilter 表）中，Pod 网络命名空间创建后，需要先写入规则，再启动业务容器。若业务先启动，其发起的连接已经通过内核路由，但若规则尚未写入，则不会被劫持，导致流量直接发出，sidecar 无法感知。因此使用 init 容器（k8s 中）预先执行 iptables 规则写入脚本，或依赖 sidecar 容器的生命周期钩子。更本质的是：规则写入是『配置』，应用启动是『运行』，必须保证配置先于首次数据包生成。
与前端已有概念的对比：这类似于前端中『事件监听器必须在事件触发前注册』。若你有一个事件总线，在发布事件之前未订阅，该事件就会丢失。iptables 规则相当于订阅注册，应用发出的第一个请求相当于事件发布。前端通过模块加载顺序保证监听器先注册，服务网格则通过 init 容器或 sidecar 启动顺序保证规则先写入。此外，iptables 规则具有『全局性』，作用于所有匹配流量，这类似于前端中代理（Proxy）或中间件（Middleware）的全局拦截，但底层是内核态实现的，而不是用户态代码。

### 3. 基础代码与实战验证
以下是一个极简的验证脚本，模拟服务网格中 sidecar 劫持与启动顺序的核心逻辑（使用 bash + iptables 命令，不依赖任何框架）：

```bash
#!/bin/bash
# 模拟 sidecar 劫持：在一个新的网络命名空间中执行
# 1. 创建网络命名空间和 veth 对，并配置 IP（省略）
# 2. 写入 iptables 规则：将所有 TCP 80 端口流量重定向到 15001
#    这里使用了 REDIRECT，本质是 DNAT 到本地 loopback 端口
iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 15001
iptables -t nat -A OUTPUT -p tcp --dport 80 -j REDIRECT --to-port 15001
# 注意：OUTPUT 链用于劫持本机应用发出的流量，PREROUTING 用于劫持外部进入的流量

# 3. 启动 sidecar 代理（监听 15001），用 nc 模拟
tcpserver -l 15001 -c 1 echo 'sidecar received' &
# 等待 sidecar 启动完成
sleep 1

# 4. 启动业务应用（模拟一个 HTTP 客户端，请求 80 端口）
echo -e 'GET / HTTP/1.0\r\n\r\n' | nc 10.0.0.1 80
# 实际结果：该请求被内核重定向到 15001，由 sidecar 处理，而不是发送到真正的 10.0.0.1:80

# 验证启动顺序：若将步骤 2 放在步骤 4 之后，则应用发出的流量不会被劫持，直接走原始路由
# 这就是为什么必须使用 init 容器先写入规则。
```

关键行注释：
- `-t nat -A PREROUTING`：在 nat 表的 PREROUTING 链追加规则，处理所有进入本网络命名空间的入站包。
- `-A OUTPUT`：处理本机进程发出的包，实现出站流量劫持。
- `--to-port 15001`：将包的目标端口改写为 sidecar 监听端口，但目标 IP 被置为 loopback，所以包会进入本机协议栈，被 sidecar socket 接收。
- 实际 Envoy 中，还会使用 `-m owner --uid-owner 1337` 来避免劫持 sidecar 自身发出的流量，防止循环。

若无法在真实环境中执行，可以使用文字化伪代码步骤：
1. 创建命名空间 ns1，配置 veth0。
2. 执行 iptables-restore 加载规则。
3. 在 ns1 中启动 sidecar 进程，监听 15001。
4. 启动业务进程，尝试连接到任意 IP 的 80 端口。
5. 观察 sidecar 日志，确认连接被劫持。
6. 反转步骤 2 和 4，观察业务进程直接连接目标，sidecar 无日志，证明顺序影响。

### 4. 常见误区与进阶思考
常见误区 1：认为 iptables 劫持是『旁路』或『镜像』。实际上它是在内核中直接改写数据包目标，属于『在线拦截』，所有流量都必须经过 sidecar，并非旁路。若规则错误，会直接导致流量丢失或循环。
常见误区 2：忽略 owner 模块的排除规则，导致 sidecar 自己发出的流量再次被劫持，形成无限递归。正确做法是使用 `-m owner ! --uid-owner <sidecar 用户>` 跳过 sidecar 进程自身的流量。
进阶思考题：在 Kubernetes 中，若 sidecar 容器与业务容器共享同一个网络命名空间，且 iptables 规则由 init 容器写入，但 init 容器退出后其 iptables 规则依然存在。请解释为什么这些规则不会随着 init 容器销毁而消失，以及如果改用 sidecar 容器启动后写入规则，会如何影响业务容器首次连接？这需要你理解 iptables 规则存储于内核网络命名空间而非进程，以及进程生命周期与内核资源的关系。
