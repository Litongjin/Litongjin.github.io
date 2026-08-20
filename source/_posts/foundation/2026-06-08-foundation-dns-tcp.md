---
title: "每日基础技术总结 · 2026-06-08 · DNS 截断响应与 TCP 回退"
date: 2026-06-08 08:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-06-08 · DNS 截断响应与 TCP 回退

## 📚 今日主题

> **DNS 截断响应与 TCP 回退**（网络基础）

### 1. 核心概念速览
DNS截断响应（Truncated Response）指DNS服务器在响应长度超出传输链路可承载的UDP报文大小限制时，将响应标志位中的TC（Truncation）位置1，并仅返回头部（无完整答案），以通知客户端改用TCP重新查询的机制。TCP回退（TCP Fallback）是客户端检测到TC位后，通过TCP协议（端口53）重新发送原始查询以获得完整响应的过程。它解决的是UDP数据报在大小受限时无法承载完整DNS消息的问题，本质是DNS协议在不可靠、无连接的UDP与可靠、流式的TCP之间的自适应切换。该机制位于DNS协议栈的传输层适配层，属于网络基础设施的关键故障恢复路径。专业工程师必须掌握它，因为任何递归解析器、CDN调度、服务发现系统都可能触发该路径，且它涉及报文格式、协议标志位、MTU、重试逻辑等多层知识，是排查DNS疑难问题的基础。

### 2. 底层原理剖析
DNS最初设计为使用UDP承载，默认最大报文长度为512字节（RFC 1035）。当响应数据超过该限制时，如果服务器不进行截断，IP层会对UDP报文进行分片，但分片在穿越NAT或防火墙时常常被丢弃，导致整体传输失败。因此协议规定：服务器必须截断响应，设置TC位，丢弃超出部分，通常只保留可放入512字节的头部及若干资源记录。客户端收到后，通过解析标志字段中的TC位（bit 9，掩码0x0200）判断是否截断。若TC=1，客户端应丢弃当前响应，并通过TCP向同一DNS服务器重新发送完全相同的查询。TCP上每个DNS消息以两字节的长度前缀标识，因此可传输任意大小的消息，无截断问题。

机制流程：
1. 客户端构造DNS查询，通过UDP发送至服务器53端口。
2. 服务器生成完整响应，若长度超过512字节（且客户端未用EDNS0通告更大缓冲），则设置TC=1，只返回头部（或部分记录）。
3. 客户端检查TC标志，若置位，则建立到服务器53端口TCP连接。
4. 客户端将原始查询封装为2字节长度+查询数据，通过TCP发送。
5. 服务器收到后，通过TCP返回完整响应（同样带长度前缀）。
6. 客户端读取长度前缀，按流方式读满整个消息。

伪代码：
response = udp_send(query)
if response.flags.TC == 1:
    tcp_connection = connect(server, 53)
    tcp_send(tcp_connection, length_prefix + query)
    length = read_uint16(tcp_connection)
    full_response = read_exactly(tcp_connection, length)

与前端已有概念的对比：该机制类似于前端存储中，当localStorage（同步、容量有限）无法容纳数据时，应用切换到IndexedDB（异步、容量更大）。两者的本质都是'在当前介质容量受限时，降级/升级到另一种更重但无限制的传输或存储方式'。但区别在于：localStorage到IndexedDB是应用层主动选择，而DNS的UDP到TCP回退是协议规定的被动响应；且TCP回退涉及连接建立、长度前缀等额外开销，正如IndexedDB需要异步API和事务开销。另外，前端工程师熟悉HTTP中基于URL长度的GET到POST切换，但那是语义改变，而DNS回退保持查询语义不变，仅改变传输层协议。

### 3. 基础代码与实战验证
```text
import socket, struct, random

def build_query(domain):
    # 构造DNS查询：随机ID + 标准标志(0x0100) + 1个问题 + 无附加
    header = struct.pack('>HHHHHH', random.randint(0, 65535), 0x0100, 1, 0, 0, 0)
    qname = b''.join(bytes([len(p)]) + p.encode() for p in domain.split('.')) + bytes([0])
    return header + qname + struct.pack('>HH', 1, 1)  # A记录, IN类

def get_tc_flag(resp):
    # 响应头部第2字节为标志位，TC位是bit 9（掩码0x0200）
    flags = struct.unpack('>H', resp[2:4])[0]
    return (flags & 0x0200) != 0

domain = 'example.com'
query = build_query(domain)

# 第一步：UDP查询
udp = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
udp.settimeout(2)
udp.sendto(query, ('8.8.8.8', 53))
resp, _ = udp.recvfrom(65535)  # 使用足够大的缓冲区，完整接收UDP响应
print('TC flag:', get_tc_flag(resp))

# 第二步：如果TC=1，执行TCP回退
if get_tc_flag(resp):
    tcp = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    tcp.connect(('8.8.8.8', 53))
    # TCP上的DNS消息前需加2字节长度前缀（大端序）
    tcp.sendall(struct.pack('>H', len(query)) + query)
    # 先读取响应长度前缀
    len_bytes = tcp.recv(2)
    resp_len = struct.unpack('>H', len_bytes)[0]
    # 再读取完整响应体（TCP是流，需循环直到读满）
    full_resp = b''
    while len(full_resp) < resp_len:
        full_resp += tcp.recv(resp_len - len(full_resp))
    print('TCP response size:', len(full_resp))
    tcp.close()
udp.close()
```

### 4. 常见误区与进阶思考
误区1：认为启用EDNS0后，UDP响应就一定不会被截断。实际上EDNS0只是让客户端通告更大的UDP缓冲（如4096），但路径MTU仍可能限制UDP报文的大小；如果中间设备阻止IP分片，服务器仍会设置TC位，客户端仍需TCP回退。
误区2：在TCP回退时，直接复用UDP发送的原始报文而忘记添加两字节长度前缀。TCP上的DNS消息必须有长度前缀，否则服务器无法区分消息边界；同样，读取响应时也必须先读长度前缀，再按该长度读取完整响应。
思考题：如果客户端在初始UDP查询中已经携带了EDNS0选项，并且服务器也支持EDNS0，但响应仍然被截断（TC=1），此时客户端应直接回退TCP，还是先尝试用更大缓冲值的EDNS0重试UDP？请结合RFC 6891和路径MTU发现机制分析两种策略的优劣，并解释为什么大多数现代递归解析器（如BIND、Unbound）在遇到TC=1时选择直接回退TCP。
