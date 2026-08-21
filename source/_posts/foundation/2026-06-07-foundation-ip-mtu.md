---
title: "每日基础技术总结 · 2026-06-07 · IP 分片与路径 MTU 发现"
date: 2026-06-07 08:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-06-07 · IP 分片与路径 MTU 发现

## 📚 今日主题

> **IP 分片与路径 MTU 发现**（网络基础）

### 1. 核心概念速览
IP 分片是 IPv4 协议在数据链路层 MTU（最大传输单元）限制下，将超过 MTU 的 IP 数据报拆分为多个较小分片传输，并在接收端重组的过程。其本质是网络层对链路层承载能力的适配机制，解决“数据报尺寸与链路 MTU 不匹配”的问题。分片由发送端或中间路由器执行，重组仅在最终目的主机完成。IPv6 中分片仅允许源节点执行，中间路由器不再分片。路径 MTU 发现（PMTUD）则通过利用 ICMPv6 的“Packet Too Big”或 IPv4 的“Fragmentation Needed”差错消息，动态探测源到目的之间最小 MTU，从而避免分片，提升传输效率与可靠性。该知识点位于 TCP/IP 协议栈的网络层与链路层交界处，是理解 TCP 分段、UDP 传输、VPN 隧道、Overlay 网络等高级机制的基础。专业工程师必须掌握，因为分片导致的安全漏洞（如 Ping of Death）、性能问题（分片丢失导致整体重传）以及 PMTUD 失效引发的黑洞问题，直接决定大规模网络应用与后端服务的稳定性与吞吐量。

### 2. 底层原理剖析
IP 分片的底层机制：发送端在构造 IP 数据报时，若总长度超过出接口 MTU，则按 MTU 进行切分。每个分片包含 IP 首部（其中 Identification 字段相同，Fragment Offset 表示该分片在原始数据报中的偏移量，单位为 8 字节，More Fragments 标志指示是否还有后续分片），数据部分被切分为若干片段。接收端根据 Identification、源/目的地址和协议号识别属于同一数据报的分片，依据 Fragment Offset 和 MF 标志按序重组。重组时若任一分片丢失或超时，则丢弃整个数据报。PMTUD 的流程：源主机先将 IP 首部的 DF（Don't Fragment）标志置 1 发送，若路径上某路由器因 MTU 限制无法转发，则丢弃该报文，并返回 ICMPv6 类型 2 的“Packet Too Big”或 ICMPv4 类型 3 代码 4 的“Fragmentation Needed”，其中携带该路由器出接口的 MTU 值。源主机据此更新自己的路径 MTU 估计值，并重新发送较小报文，直到不再收到差错消息。该过程可类比前端领域中的“接口契约演化”：IP 分片类似于通过不同大小的 chunk 传输同一资源，而 PMTUD 类似于内容协商（Content Negotiation）中的 Accept-* 头与服务器响应 406 后客户端调整请求，最终达成双方可接受的资源尺寸。区别在于 IP 分片是网络层强制适配，对上层透明；PMTUD 是源主机主动探测，依赖 ICMP 回显，且需要路径上的防火墙/路由器允许 ICMP 差错消息通过。

### 3. 基础代码与实战验证
由于 IP 分片与 PMTUD 是内核协议栈行为，不依赖用户态库，以下用 Python + raw socket 构造带 DF 标志的 UDP 报文，并捕获 ICMP 差错消息，验证 PMTUD 的基本流程。

```python
import socket, struct, time

def checksum(data):
    # 计算 Internet 校验和：每 16 位取反求和，再取反
    s = 0
    for i in range(0, len(data), 2):
        if i + 1 < len(data):
            s += (data[i] << 8) + data[i+1]
        else:
            s += data[i] << 8
    while s >> 16:
        s = (s & 0xFFFF) + (s >> 16)
    return (~s) & 0xFFFF

def build_ip_header(total_len, frag_offset, df):
    # IP 首部固定 20 字节；flags 中 DF=0x4000，MF=0x2000，frag_offset 低 13 位
    flags_frag = frag_offset | (0x4000 if df else 0)
    return struct.pack('!BBHHHBBH4s4s',
                       0x45, 0, total_len, 0x1234, flags_frag,
                       64, 17, 0,  # TTL=64, 协议=17(UDP), 校验和先置0
                       socket.inet_aton('192.168.1.100'), socket.inet_aton('8.8.8.8'))

# 创建原始 socket，仅用于发送 IP 报文（需 root 权限）
send_sock = socket.socket(socket.AF_INET, socket.SOCK_RAW, socket.IPPROTO_RAW)
# 创建 ICMP 监听 socket 接收错误报文
icmp_sock = socket.socket(socket.AF_INET, socket.SOCK_RAW, socket.IPPROTO_ICMP)

payload = b'A' * 1500  # 超过典型以太网 MTU 1500，但 IP 首部 20 字节+UDP 8 字节导致总长 1528
udp_len = 8 + len(payload)
udp_header = struct.pack('!HHHH', 12345, 53, udp_len, 0)  # 源端口 12345, 目的 53
udp_packet = udp_header + payload

ip_total_len = 20 + udp_len
ip_header = build_ip_header(ip_total_len, 0, df=True)  # DF=1 禁止分片

# 填充 IP 校验和
ip_header = ip_header[:10] + struct.pack('!H', checksum(ip_header)) + ip_header[12:]

send_sock.sendto(ip_header + udp_packet, ('8.8.8.8', 0))

# 等待 ICMP 报文，最多 5 秒
icmp_sock.settimeout(5)
try:
    data, addr = icmp_sock.recvfrom(65535)
    # 解析 ICMP 首部：类型 3（目的不可达）代码 4（需要分片但 DF 置位）
    icmp_type, icmp_code = data[20], data[21]
    if icmp_type == 3 and icmp_code == 4:
        # 从 ICMP 负载中提取原始 IP 首部，再读取下一跳 MTU（ICMP 后 2 字节）
        mtu = struct.unpack('!H', data[6:8])[0]
        print(f'PMTUD 收到 ICMP 差错: 下一跳 MTU = {mtu}')
    else:
        print(f'收到类型 {icmp_type} 代码 {icmp_code}')
except socket.timeout:
    print('超时：可能路径 MTU 足够大或 ICMP 被丢弃')
```

实际中，多数系统已实现 PMTUD，应用层只需在 UDP 中使用 `IP_MTU_DISCOVER` 套接字选项（Linux）或 `setsockopt` 控制 DF 行为；TCP 则自动执行 PMTUD。本代码演示了手工构造 IP 报文并监听 ICMP 的核心验证步骤。

### 4. 常见误区与进阶思考
误区一：认为“分片对上层透明，因此无需关注”。实际上，分片会增加接收端重组开销，且任一网络设备丢弃一个分片就会导致整个数据报重传，对 UDP 这类无重传机制的应用可能直接丢失。更危险的是，分片重组时的偏移量校验缺陷曾导致 Ping of Death 漏洞，且当前网络中的中间盒（如 NAT、防火墙）常丢弃分片，导致实际传输失败。

误区二：认为“设置了 DF 位就一定能避免分片”。PMTUD 依赖 ICMP 差错消息，若路径上的路由器或防火墙屏蔽了 ICMP（尤其 ICMPv6 类型 2 或 ICMPv4 类型 3/4），源主机永远收不到反馈，就会持续重传超大报文，造成“PMTUD 黑洞”。此时 TCP 可能退化为 MSS 协商，但 UDP 等无连接协议则无解。

深度思考题：假设你在一个 IPv4 网络中，客户端通过 TCP 与服务器通信，TCP 握手时双方均通告 MSS=1460，但实际路径 MTU 只有 1280。请推导：TCP 分段后每个段长度为 1460，IP 层必须分片，而 DF 位默认为 1 时，第一次发送会触发 PMTUD 得到 MTU=1280。随后 TCP 会根据 ICMP 消息调整 MSS 为 1240（1280-20-20）。然而，如果路径上某个中间路由器仅针对 IP 分片（而非整条路径）返回 ICMP 消息，且该 ICMP 消息的源地址是路由器内网地址，而客户端所在防火墙只允许从已知服务器 IP 进入的 ICMP，那么该 ICMP 会被丢弃。请问此时 TCP 连接会出现什么现象？如何从客户端/服务端两侧验证并解决？
