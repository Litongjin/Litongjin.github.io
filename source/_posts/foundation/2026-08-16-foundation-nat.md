---
title: "每日基础技术总结 · 2026-08-16 · 网络地址转换 NAT 与穿透"
date: 2026-08-16 08:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-08-16 · 网络地址转换 NAT 与穿透

## 📚 今日主题

> **网络地址转换 NAT 与穿透**（网络基础）

### 1. 核心概念速览
NAT（Network Address Translation）是位于IP层与传输层之间的地址/端口重写机制，本质是网关设备对经过的数据包头部（源/目的IP、源/目的端口、校验和）进行有状态改写，以在私有地址域与公网地址域之间建立映射。它解决的问题是IPv4地址短缺，以及作为边界防火墙隐藏内网拓扑。其核心机制是维护一张映射表（会话表），记录内网主机(ip, port)与公网(ip, port)的对应关系，并据此反向改写回程流量。NAT在整个网络体系中处于网络层与传输层交界处，是理解端到端连通性、P2P通信、防火墙策略、QUIC/HTTP3连接迁移等现代网络架构的基础。专业工程师必须掌握它，因为现代云原生环境（K8s Service、Docker bridge网络、负载均衡器、服务网格）和实时通信（WebRTC、VoIP）都隐式依赖NAT行为，不理解其映射语义就无法调试连接问题、设计穿透方案或评估网络性能。

### 2. 底层原理剖析
NAT的底层机制分为三类基本行为：1）Full Cone（完全圆锥）：内部主机(ip:port)首次出向包建立映射后，外部任意主机均可通过该公网映射向内部主机发包；2）Restricted Cone（限制圆锥）：仅内部主机曾向其发送过包的外部IP可以回包；3）Port Restricted Cone（端口限制圆锥）：进一步限制外部端口也必须匹配。第四类为Symmetric NAT（对称NAT）：同一内部(ip:port)发往不同目的(ip:port)会建立不同的公网映射，且只接受之前通信过的对端回包。前三种统称Cone NAT，映射与目的无关；Symmetric NAT映射与目的地址强相关。

核心流程：内网主机A(10.0.0.2:5000)向公网服务器B(8.8.8.8:443)发送SYN，NAT设备改写源地址为公网IP(203.0.113.1)，并分配新源端口如40000，记录映射项：{10.0.0.2:5000 <-> 203.0.113.1:40000, 对端:8.8.8.8:443}。回包到达NAT设备时，根据(目的IP:端口=203.0.113.1:40000)查找映射表，改写成(10.0.0.2:5000)后转发。如果使用Cone NAT，映射不依赖对端地址，因此任意对端只要发往该公网IP:端口即可被转发；而Symmetric NAT在A向B发包时生成的映射只允许B回包，且如果A向另一个对端C发包，会生成另一个不同的公网端口映射。

穿透原理：当两个内网主机A和B都在NAT后面时，双方都无法直接通过对方内网IP访问。穿透思路是：双方先通过公网信令服务器(S)交换各自当前的公网IP:端口（即NAT映射地址）。随后双方同时向对方的公网映射地址发送UDP包，触发各自NAT设备建立/更新出向映射（打洞）。由于Cone NAT的映射具有通用性，此时双方发出的包都能被对端NAT接受并转发，从而建立双向通信。Symmetric NAT因为映射与目的绑定，单纯打洞无效，需要更高级的端口预测或TCP中继。

与前端概念的对比：NAT映射表类似于前端JavaScript中对象（Map）的键值对，但键是四元组（源IP、源端口、目的IP、目的端口），值是对应的重写规则。TypeScript的接口（interface）是编译期的结构契约，运行时不保留；NAT是运行时的动态状态，会随连接生命周期变化。前端中的“端口”通常指应用层抽象（如HTTP服务的80端口），而NAT的端口是传输层的16位标识，两者不在同一语义层。NAT更接近HTTP反向代理的虚拟主机机制：反向代理根据Host头路由到不同内部服务，NAT根据端口映射路由到不同内网主机，但NAT工作在更底层，无需应用层解析。

### 3. 基础代码与实战验证
以下用Python模拟NAT映射表与包重写过程（非真实网络栈），用于验证Cone NAT与Symmetric NAT的差异。

```python
import hashlib

class NATDevice:
    def __init__(self, public_ip):
        self.public_ip = public_ip
        self.map = {}          # key: (internal_ip, internal_port), value: list of mapping entries
        self.next_port = 40000
        self.type = 'cone'     # 或 'symmetric'

    def translate_outbound(self, src_ip, src_port, dst_ip, dst_port):
        key = (src_ip, src_port)
        if self.type == 'symmetric':
            # 对称NAT：每个(内网ip,端口,目的ip,端口)生成独立公网端口
            map_key = (key, dst_ip, dst_port)
            if map_key not in self.map:
                public_port = self.next_port; self.next_port += 1
                self.map[map_key] = (self.public_ip, public_port, dst_ip, dst_port)
                print(f"[SYMMETRIC] 新建映射: {src_ip}:{src_port} -> {self.public_ip}:{public_port} (目的 {dst_ip}:{dst_port})")
            else:
                print(f"[SYMMETRIC] 使用已有映射: {src_ip}:{src_port} -> {self.map[map_key]}")
            entry = self.map[map_key]
            # 重写包源地址
            return (entry[0], entry[1])
        else:
            # Cone NAT：只要内网ip和端口相同，就复用同一公网端口，与目的无关
            if key not in self.map:
                public_port = self.next_port; self.next_port += 1
                self.map[key] = (self.public_ip, public_port)
                print(f"[CONE] 新建映射: {src_ip}:{src_port} -> {self.public_ip}:{public_port}")
            else:
                print(f"[CONE] 复用映射: {src_ip}:{src_port} -> {self.map[key]}")
            return self.map[key]

    def translate_inbound(self, dst_ip, dst_port, src_ip, src_port):
        # 回包查找：根据公网目的IP和端口找到内网映射
        if self.type == 'symmetric':
            for (key, (pub_ip, pub_port, orig_dst_ip, orig_dst_port)) in self.map.items():
                if pub_port == dst_port and orig_dst_ip == src_ip and orig_dst_port == src_port:
                    return (key[0], key[1])
            return None
        else:
            for key, (pub_ip, pub_port) in self.map.items():
                if pub_port == dst_port:
                    return (key[0], key[1])
            return None

# 验证：内网主机 10.0.0.2:5000 先向 8.8.8.8:443 发包，再向 9.9.9.9:53 发包
nat = NATDevice('203.0.113.1')
nat.type = 'cone'
print("=== Cone NAT 行为 ===")
nat.translate_outbound('10.0.0.2', 5000, '8.8.8.8', 443)
nat.translate_outbound('10.0.0.2', 5000, '9.9.9.9', 53)  # 复用同一公网端口，所以外部任何主机都能用该端口回包

nat2 = NATDevice('203.0.113.2')
nat2.type = 'symmetric'
print("\n=== Symmetric NAT 行为 ===")
nat2.translate_outbound('10.0.0.2', 5000, '8.8.8.8', 443)
nat2.translate_outbound('10.0.0.2', 5000, '9.9.9.9', 53)  # 新公网端口，导致外部无法复用之前的映射
```

关键说明：第23行和第33行分别体现了两种NAT的映射键设计——Symmetric以(内网IP,端口,目的IP,端口)为键，Cone以内网(IP,端口)为键。这直接决定了回包转发的可达性范围。真实网络中，NAT设备还会改写校验和（IP/TCP/UDP checksum），并维护超时时间（UDP通常30秒~5分钟），这些代码未体现，但映射表查找逻辑一致。

若无法运行Python，可理解伪代码：出向包到达NAT -> 根据类型生成映射键 -> 查映射表，无则分配新公网端口并记录，有则复用 -> 修改源IP和端口 -> 更新校验和 -> 转发；回向包到达NAT -> 根据公网目的IP/端口（及Symmetric下的源IP/端口）查表 -> 若命中则改写目的IP/端口并转发，否则丢弃。

### 4. 常见误区与进阶思考
误区1：认为NAT只是IPv4地址节省工具，而忽略其安全隔离和状态感知语义。实际上NAT是一种有状态防火墙，它天然阻断外部主动发起的连接（除非静态映射或UPnP端口转发）。很多工程师在调试WebRTC或P2P时，只关注STUN拿到的公网地址，却忽略自身NAT类型对打洞成功率的决定性影响。若NAT是Symmetric，即使STUN反射地址正确，直接P2P也会失败，必须使用TURN中继或端口预测。

误区2：混淆NAT与代理（Proxy）的工作层次。NAT在网络层/传输层透明重写包，不感知应用层内容；而代理（如HTTP代理、SOCKS5）是应用层/传输层的中间人，必须显式配置客户端。前端工程师常犯的认知是把Kubernetes Service的NodePort或负载均衡器的端口转发与NAT等同。实际上K8s kube-proxy的DNAT（Destination NAT）确实是一种NAT（修改目的IP/端口），但纯NAT不修改HTTP Host头、不解析TLS SNI，而L7代理（如Ingress NGINX）会解析应用层。这导致排查问题方式不同：NAT问题通常在包捕获中看IP/端口改写，代理问题则要关注HTTP头或TLS握手。

思考题：假设你在一个Full Cone NAT后面，通过STUN服务器发现自己的公网IP:端口为203.0.113.1:40000。此时另一个位于Symmetric NAT后面的对端（公网IP为198.51.100.2:50000）向你发送UDP包，你是否能收到？为什么？如果换过来，你向该对端发送UDP包，对端能否收到？请从两种NAT的映射表查找条件出发，分别分析收发路径上每个NAT的行为，并解释为什么在P2P UDP打洞中，至少一方需要是Cone NAT才能成功。
