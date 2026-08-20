---
title: "每日基础技术总结 · 2026-07-08 · CNI 插件链：ADD/DEL 操作与 bridge 插件的 IPAM"
date: 2026-07-08 08:00:00
categories: [技术分享]
tags: ["技术分享", "DevOps 与云原生"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-08 · CNI 插件链：ADD/DEL 操作与 bridge 插件的 IPAM

## 📚 今日主题

> **CNI 插件链：ADD/DEL 操作与 bridge 插件的 IPAM**（DevOps 与云原生）

### 1. 核心概念速览
CNI（Container Network Interface）是容器运行时与网络插件之间的一套契约规范，定义了如何为容器创建、销毁网络。其本质是一个进程级协议：运行时通过环境变量传入操作命令（ADD/DEL/CHECK 等），并通过标准输入传递网络配置 JSON，插件执行后通过标准输出返回结果。插件链则是指多个 CNI 插件按顺序执行形成管道，例如先调用 ipam 插件分配 IP，再调用 bridge 插件配置网桥与 veth 对，前一个插件的输出可以作为后一个插件的输入。ADD 操作负责将容器加入网络，DEL 操作负责回收网络资源。bridge 插件是 CNI 中最基础的插件之一，它利用 Linux 内核的 bridge 模块创建网桥，并通过 veth pair 将容器网络命名空间与主机连接。IPAM（IP Address Management）插件（如 host-local）负责从指定子网中分配唯一的 IP 地址，并管理租约。该体系解决的核心问题是：在云原生（尤其是 Kubernetes）环境中，如何以可插拔、标准化方式管理异构容器网络。它位于容器运行时（如 containerd）与内核网络栈之间，是容器网络虚拟化的基座。专业工程师必须掌握，因为它是理解网络插件（Calico、Flannel、Cilium）工作原理的基础，也是排查容器网络故障的根本线索。

### 2. 底层原理剖析
CNI 插件的调用机制是严格的进程级接口。运行时设置环境变量 CNI_COMMAND（ADD/DEL/CHECK/VERSION）、CNI_CONTAINERID、CNI_NETNS、CNI_IFNAME、CNI_ARGS 等，将网络配置 JSON（包含 cniVersion、name、plugins 或 type、ipam 等字段）写入标准输入。插件进程只存活一次调用，不驻留。插件链通过运行时依次调用列表中的每个插件，前一个插件的结果作为下一个插件的输入参数（或通过缓存传递），最终汇总返回。

bridge 插件的 ADD 流程本质如下：
1. 检查或创建网桥（bridge）设备，确保网桥存在。
2. 创建 veth pair，一端放入容器网络命名空间（CNI_NETNS），重命名为 CNI_IFNAME，另一端附着到网桥。
3. 调用 IPAM 插件（如 host-local）从配置的 subnet 中分配一个 IP 地址，返回 IP、网关、路由等信息。
4. 在容器网络命名空间内设置 IP 地址，并添加默认路由（或指定路由）。
5. 输出 CNI 结果 JSON（包括 IP、MAC、接口等）。

DEL 流程则逆序：清理由 IPAM 分配的 IP（释放租约），删除 veth pair，可选删除网桥（若不再使用）。

IPAM 插件本质是一个地址分配器。host-local 通过文件锁和文件存储记录已分配地址，确保并发安全。分配策略通常是线性扫描子网中的地址，跳过已被占用的。地址的释放就是删除文件中的记录。

与前端知识体系的对比：CNI 接口类似于 Java 中的接口——实现必须严格遵循签名（环境变量、stdin/stdout），是编译期约束；而 TypeScript 的接口是结构类型，只要形状匹配即可，更灵活。插件链类似于 Express 中间件或 webpack loader 的管道，但更底层——每个插件是独立进程，通过标准 I/O 通信，而非函数调用。理解这一点，能帮助你区分“协议契约”和“编程接口”的差异。

### 3. 基础代码与实战验证
以下是一个极简的 CNI 插件示例（Python 模拟），仅演示 ADD 和 DEL 的核心逻辑，以及 IPAM 分配的本质。实际插件用 Go/C 实现，但协议相同。

```python
import os, sys, json, ipaddress

# 从标准输入读取网络配置（JSON）
config = json.load(sys.stdin)
# 从环境变量获取操作命令和容器信息
command = os.environ['CNI_COMMAND']
container_id = os.environ['CNI_CONTAINERID']
netns = os.environ['CNI_NETNS']

# IPAM 状态存储（模拟 host-local 文件）
IP_FILE = '/tmp/cni-ipam-' + container_id + '.json'

def allocate_ip(subnet):
    # 实际中需要原子操作，这里简化为静态分配
    net = ipaddress.ip_network(subnet)
    # 分配第一个可用 IP（实际会维护分配表）
    ip = list(net.hosts())[0]  # 伪代码，应检查冲突
    # 将分配记录写入文件，以便 DEL 时释放
    with open(IP_FILE, 'w') as f:
        json.dump({'ip': str(ip)}, f)
    return ip

def release_ip():
    # 删除分配记录
    if os.path.exists(IP_FILE):
        os.remove(IP_FILE)

def bridge_add(ip):
    # 底层调用 ip link add, ip addr add 等命令
    # 这里仅输出模拟结果
    print(json.dumps({
        'cniVersion': '0.3.1',
        'ips': [{'version': '4', 'address': f'{ip}/24', 'gateway': '10.22.0.1'}]
    }))

def bridge_del():
    # 清理 veth、网桥等（省略）
    pass

if command == 'ADD':
    # 1. 从配置中读取 IPAM 子网
    subnet = config['ipam']['subnet']
    # 2. 分配 IP
    ip = allocate_ip(subnet)
    # 3. 配置 bridge（创建 veth，设置 IP，路由）
    bridge_add(ip)
elif command == 'DEL':
    # 1. 释放 IP
    release_ip()
    # 2. 清理网络设备
    bridge_del()
```

这个脚本体现了 CNI 插件的协议本质：通过环境变量和标准输入接收参数，通过标准输出返回结果。IPAM 的分配与释放直接映射到文件系统操作，与 host-local 的原理一致。

### 4. 常见误区与进阶思考
误区一：误认为 CNI 插件是长期运行的守护进程或服务。实际上每次 ADD/DEL 都会重新启动一个进程，插件不能依赖内存状态，必须通过外部持久化（如文件、etcd）来保存状态。否则容器重启后无法正确恢复网络。

误区二：混淆 IPAM 与网络配置。IPAM 只负责分配/回收 IP 地址，不负责设置网桥、路由、iptables 等。bridge 插件才是真正执行网络配置的实体。理解这个边界，才能正确设计插件链的职责划分。

思考题：在 Kubernetes 中，一个 Pod 有多个容器共享网络命名空间。当 Pod 被删除时，CNI 的 DEL 操作会被调用多次（对每个容器接口）。如何保证 DEL 的幂等性？如果 ADD 成功后，DEL 调用失败，导致 IP 未释放，下一次 ADD 是否会分配重复 IP？请从 CNI 规范中的幂等要求和 IPAM 的状态设计角度分析。
