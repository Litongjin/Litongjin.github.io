---
title: "每日基础技术总结 · 2026-09-26 · IP 分片与路径 MTU 发现（PMTUD）黑洞"
date: 2026-09-26 07:05:30
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-26 · IP 分片与路径 MTU 发现（PMTUD）黑洞

## 📚 今日主题

> **IP 分片与路径 MTU 发现（PMTUD）黑洞**（网络基础）

### 1. 核心概念速览
IP 分片是将超过链路 MTU 的 IP 数据报切分为多个片段独立传输、在接收端重组的过程，本质上是网络层对二层链路载荷上限的适配机制。路径 MTU 发现（PMTUD）则是通过探测源端到目的端整条路径上最小 MTU 值来避免分片的机制。PMTUD 黑洞指路径中某设备丢弃需要分片的数据报却未返回 ICMP 差错消息（通常因防火墙/安全策略过滤了 ICMP type 3 code 4），导致源端一直重传超大报文、连接无法建立或数据传输停滞。它位于网络层与传输层的交界面，直接决定 TCP MSS 协商、UDP 大包发送策略和隧道封装设计；专业工程师必须掌握它，因为凡是涉及 VPN、隧道、云网络、容器网络、移动网络或跨广域网传输的现代分布式系统，都会因 PMTUD 黑洞出现看似无解的超时、假死和性能断崖，而问题的根源往往在最基础的 IP 分片语义和 ICMP 约束上，不掌握这一点便无法建立对端到端包传输路径的整体心智，排查问题只能靠修参数而非看本质。

### 2. 底层原理剖析
IP 分片的底层语义：IPv4 首部中的 Identification（16 位）、Flags（DF/MF）、Fragment Offset（13 位，以 8 字节为单位）共同完成分片与重组。发送端对大于 MTU 且 DF=0 的报文，按 8 字节对齐切分，每片独立路由与传输；接收端依据源 IP、目的 IP、Identification 和协议号分组，按 Fragment Offset 排序，在 MF=0 的片到达且所有间隙填补后重组。分片不保证按序到达，且一旦任意一片丢失，整包作废，因此实时性差、头部开销倍增。

PMTUD 机制：源端发送 DF=1 且大小等于假设 MTU 的探测包；路径中若某链路 MTU 小于包长，路由器无法转发又不能分片时，丢弃包并回送 ICMP type 3 code 4（Fragmentation Needed），同时附带上该链路 MTU 值（ICMP 扩展中的 Next-Hop MTU 字段）。源端据此降低 MTU 并重发，直至探测成功。IPv6 中路由器不做分片，源端执行同样的路径 MTU 探测，过程一致。

黑洞原理：若路径上的中间设备（常为防火墙或安全组）丢弃 DF=1 的超大包，且出于安全策略丢弃或限速了 ICMP type 3 code 4 报文，源端因永远收不到反馈而持续重传原始大小报文，探测永不收敛，表现为 TCP 会话卡在 SYN（SYN 包未带数据时利用 MSS 协商，但后续大数据段被黑洞）或 UDP/SCTP 数据整块超时。TCP 的 MSS 协商只能在握手时避免初始报文过大，但若隧道/封装导致最终 IP 包超出路径 MTU，且 MSS 未按实际隧道开销调整，仍会触发 PMTUD 黑洞。

本质对比：前端工程师熟悉的 TS interface 与 Java interface 的差异是“结构性类型系统 vs 名义类型系统”的编译期契约差异；而 IP 分片与 PMTUD 是运行期网络协议状态机，前者是静态约定、后者是动态探测协议。更贴切的桥梁是 HTTP 的 Content-Length 与 chunked 传递——分片类似将资源切成多个 chunk 且不要求 chunk 按序到达，但接收者必须完整重组后才可交付上层；PMTUD 类似 Web 的 413 响应后客户端自动调整请求体大小重试，但要求 413 反馈和客户端重试逻辑都畅通，一旦反馈被中间代理吞掉，就是黑盒超时。

关键状态流程（伪代码）：
```
发送端: data = upper_layer_pdu
path_mtu = local_mtu 或 cached
loop:
  ip_pkt = build_ip(data, df=1, size<=path_mtu)
  send(ip_pkt)
  if receive_icmp_frag_needed(mtu_feedback):
      path_mtu = min(path_mtu, feedback_mtu)
  elif timeout:
      if df=1 and packet_size > 0:  # 黑洞可能
          retry, eventually reduce mtu manually
其他:  receive_icmp_frag_needed
  if 原包 DF=1: 缓存新 MTU 并通知传输层
  传输层: snd_mss = min(advertised_mss, path_mtu - ip_header - tcp_header)
```

### 3. 基础代码与实战验证
由于 PMTUD 涉及内核协议栈与路由器行为，这里提供一段带底层注释的伪代码，描述 TCP 连接建立后大数据段被 PMTUD 黑洞吞没的完整时序，以及检测手段；关键行以中文注释解释内核与网络设备如何协作。

```
# 阶段一：TCP 三次握手，双方 MSS 协商（基于各自所在链路 MTU）
client -> syn:  mss=1460 (假定 client 本地 MTU=1500)
server -> syn+ack: mss=1460
client -> ack: 连接建立
# 注意：此 MSS 只代表“端到端不涉及隧道/中间更低 MTU 时”的最大段大小

# 阶段二：发送大数据段（假设应用写 8KB）
tcp_segment = 8192 bytes
ip_packet = build_ip(tcp_segment, df=1, total_len=8200+20)
# 内核 TCP 通常依据 MSS 分段为 1460 字节，即 IP 包大小 1500；
# 但若中间存在隧道（如 IPsec/GRE/VXLAN），外层 IPv4 头会增加 20 或更多字节，
# 导致实际 IP 包 = 1500（内层）+ 20+8+20（外层头） > 中间链路 MTU 1500
# 此时中间路由器需要分片，但由于 DF=1，它丢弃并回送 ICMP type3 code4 并附 MTU

# 阶段三：路由器行为（设备防火墙/安全组抑制 ICMP）
if packet_need_fragment and df_set:
    drop(packet)
    send_icmp(type=3, code=4, next_hop_mtu=1400) # 但此 ICMP 被安全策略丢弃
# 客户端网卡/协议栈永远不会收到反馈

# 阶段四：客户端超时重传（TCP 指数退避，约 1,2,4,8... sec）
retransmit(ip_packet)  # 原大小不变，因为 path_mtu 未更新
# 多次重试后连接超时/重置，或应用感知到停滞

# 验证/排查：
# linux 下禁用 PMTUD 黑洞：sysctl -w net.ipv4.ip_no_pmtu_disc=0
# 设置更小 MSS：iptables -t mangle -A OUTPUT -p tcp --tcp-flags SYN,RST SYN \
#   -j TCPMSS --set-mss 1400  (本质是静态避开PMTUD失败路径)
# 主动探测路径 MTU：ping -M do -s 1472 host  (ICMP payload=1472 -> 总包1500)
```

这段代码的核心：DF=1 是 PMTUD 探测的开关，ICMP type3 code4 是唯一的反馈通道。若反馈被中间设备静默丢弃，TCP 重传逻辑只会无限重发同一大小的报文，造成黑洞。

### 4. 常见误区与进阶思考
误区一：认为 TCP MSS 协商后就不会出现 IP 分片或 PMTUD 黑洞。MSS 只在建立连接时根据两端直连链路 MTU 协商，中间路径的 MTU 更小、或存在隧道封装（VXLAN/GRE/IPsec/MPLS 叠加）时，实际 IP 包会比 MSS+40 更大，甚至超过中间链路阈值。DF=1 的机制使得报文无法被分片，只能依赖 ICMP 反馈调整，而反馈被安全组/防火墙过滤便形成黑洞。结论：MSS 只是降低概率，不能根除；必须关注端到端最小 MTU 以及隧道封装开销（额外至少有 20+8+20 或更多字节）。

误区二：反复探究“为什么 ping 能通、TCP 却超时？” 因为 ping 默认不带数据且 DF=0，128 字节的 ICMP 远小于所有链路 MTU，根本不会触发分片判定；即使 ping -s 大于 MTU，若 DF=0 会正常分片转发不会触发 ICMP。只有用 DF=1 且包长超过实际路径 MTU 才能暴露黑洞。TCP 数据包本身带 DF=1 且是大包，所以卡住；而 ping 的小包路径不受影响，这容易误导工程师去排查应用层或 DNS，而真正机制是 ICMP 被黑洞。

进阶思考：在一条物理路径中，假设中间路由器 A 的 MTU=1400，B 的 MTU=1500，源端 TCP MSS 协商为 1460，且路径中的防火墙丢弃所有 ICMP type 3。请问：若将 TCP MSS 手动固定为 1360（即避免超过 1400 的 IP 包），是否一定能避免黑洞？为什么？考虑 IPv6 场景（路由器不分片、且 MPLD 机制依赖 ICMPv6 type 2），如果中间节点对 ICMPv6 丢弃，后果是否与 IPv4 完全相同？请从协议栈各层（传输层、网络层、中间设备策略）逐层论证你的答案。
