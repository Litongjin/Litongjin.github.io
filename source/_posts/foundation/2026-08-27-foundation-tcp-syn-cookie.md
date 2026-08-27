---
title: "每日基础技术总结 · 2026-08-27 · TCP 三次握手与 SYN Cookie"
date: 2026-08-27 08:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-27 · TCP 三次握手与 SYN Cookie

## 📚 今日主题

> **TCP 三次握手与 SYN Cookie**（网络基础）

### 1. 核心概念速览
TCP 三次握手是传输控制协议（TCP）在连接建立阶段使用的状态同步机制，其本质是通过交换初始序列号（ISN）来确认通信双方的接收与发送能力，并同步各自窗口、选项等参数。它解决的核心问题是：在不可靠的IP网络之上，为可靠字节流传输建立端到端的双向有序信道。三次握手并非简单的“三次报文交换”，而是通过SYN、SYN-ACK、ACK三个段完成四个信息状态的确认：客户端的ISN、服务端的ISN、客户端具备接收能力、服务端具备接收能力。SYN Cookie是服务端在SYN Flood攻击下用于防止资源耗尽的一种无状态握手机制：服务端不分配任何传输控制块（TCB），而是将连接参数（包括ISN、时间戳、MSS等）经哈希运算编码为一个Cookie作为SYN-ACK的序列号，待客户端ACK返回时通过校验Cookie合法性来确认连接真实性，从而延迟资源分配至握手完成之后。该机制是网络协议栈抗DoS攻击的基础设计，专业工程师必须掌握其状态机转换与内核实现，因为它是理解TCP性能优化、攻击防御以及高并发连接管理（如Nginx、LVS等）的基石。

### 2. 底层原理剖析
TCP三次握手的底层机制基于有限状态机（FSM）与序列号同步。客户端初始状态CLOSED，主动调用connect后进入SYN_SENT，发送SYN段，携带客户端初始序列号ISN_c（由系统时钟+随机噪声生成）。服务端处于LISTEN状态，收到SYN后，若启用SYN Cookie则不创建半连接队列项，直接计算Cookie = hash(源IP、源端口、目的IP、目的端口、服务端秘密、t、MSS index)（实际为SHA-1等哈希编码），其中t为时间计数器。服务端回复SYN-ACK，将序列号设为Cookie值，同时携带服务端ISN_s（在SYN Cookie模式下即Cookie本身），并确认客户端ISN（ACK = ISN_c + 1）。客户端收到SYN-ACK后，验证ACK号是否等于ISN_c+1（验证服务端收到了自己的SYN），然后发送ACK段，序列号为ISN_c+1，确认号为Cookie+1（即ISN_s+1）。在此过程中，客户端不承担任何状态验证负担（仅需按常量递增），而服务端在收到最终ACK后，通过重新计算当前时间窗内的哈希值并与ACK确认号减1比较，若匹配，则构造TCB建立连接，进入ESTABLISHED。

伪代码（服务端接收SYN路径）：
```
function handle_syn(syn_pkt):
    if syn_cookie_enabled and tcb_pool_exhausted_or_over_threshold:
        # 无状态模式：不分配TCB，计算Cookie
        t = (timestamp >> 6) & MASK  # 64秒一个窗口
        cookie = sha1(src_ip, src_port, dst_ip, dst_port, secret, t, mss_idx) & 0xFFFFFFFF
        cookie |= mss_idx << 24        # 低8位存MSS索引
        send_syn_ack(seq=cookie, ack=isn_c+1)
        return
    else:
        # 传统模式：分配半连接TCB，加入SYN队列
        tcb = alloc_tcb(...)
        add_to_syn_queue(tcb, syn_pkt)
        send_syn_ack(seq=isn_s, ack=isn_c+1)
```

服务端接收ACK路径（仅当处于SYN Cookie模式时）：
```
function handle_ack(ack_pkt):
    cookie = ack_pkt.ack_num - 1  # 客户端确认的是Cookie+1
    t = (timestamp >> 6) & MASK
    if cookie_valid(cookie, t) or cookie_valid(cookie, t-1):
        mss_idx = (cookie >> 24) & 0xFF
        tcb = construct_tcb_from_cookie(cookie, ...)
        handshake_complete(tcb)
    else:
        drop_packet()
```

与前端概念的异同：类似于TypeScript的接口（interface）与Java的接口（interface）。TCP三次握手更像是TypeScript的结构化类型系统——注重最终行为（双方对序列号的确认），而不要求显式声明“我有接收能力”；SYN Cookie则类似于一个基于哈希的运行时类型检查，它在不预先分配状态的情况下，通过计算得出的签名（Cookie）在后续验证有效性，与前端中JWT（JSON Web Token）的无状态身份验证思想一致——服务端不保存会话，而是在令牌中编码验证信息。关键区别在于网络层有重传、超时、乱序等物理不确定性，而前端接口是静态契约，不存在时间窗口和DDoS攻击面。

### 3. 基础代码与实战验证
由于TCP协议栈属于操作系统内核，无法用纯用户态代码直接模拟协议栈，但可以用socket API演示三次握手与SYN Cookie的观测。以下提供两种极简代码：1. 用Python socket模拟客户端三次握手过程（观察SYN_SENT→ESTABLISHED状态）；2. 用tcpdump/netstat命令验证SYN Cookie。

代码1：Python客户端，模拟三次握手的最终ACK发送（本质上connect()函数封装了三次握手）
```python
import socket

# 创建TCP套接字，内核协议栈会完成三次握手
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
# 设置TCP_NODELAY以观察实际发送，默认即可
s.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)

# connect()内部：发送SYN -> 等待SYN-ACK -> 发送ACK，系统调用返回即建立连接
try:
    s.connect(('192.168.1.1', 8080))  # 目标为TCP监听端口
    print('连接建立，内核完成三次握手')
    # 发送数据，验证可靠字节流
    s.sendall(b'hello')  # sendall返回后，数据已进入内核发送缓冲区
except Exception as e:
    print(f'握手失败: {e}')
finally:
    s.close()  # 发送FIN，开始四次挥手
```

代码2：验证SYN Cookie的内核行为（在Linux环境下，不依赖外部库）
```bash
# 查看当前SYN Cookie是否启用（0关闭，1开启）
sysctl net.ipv4.tcp_syncookies

# 启用SYN Cookie
sysctl -w net.ipv4.tcp_syncookies=1

# 用strace观察connect系统调用时的状态（可选）
strace -e trace=connect python3 -c "import socket; s=socket.socket(); s.connect(('127.0.0.1', 8080))"

# 在服务器端开启tcpdump抓包，查看三次握手报文序列号特征
# 当SYN Cookie生效时，SYN-ACK的TCP序列号（Seq）是一个随机Cookie，而非传统的时间戳随机生成器
# 使用下述命令抓取本地环回的握手包
sudo tcpdump -i lo -nn -S 'tcp port 8080 and (tcp-syn|tcp-ack)' -v
```

代码3（文字化伪代码）：内核中SYN Cookie的生成与校验的核心算法（Linux内核实际实现为`cookie_v4_init_sequence`函数）
```c
// 伪代码，实际使用SHA1和计数抖动
#define COOKIE_BITS 24
#define MSS_SHIFT 8

u32 cookie_v4_init_sequence(struct sk_buff *skb)
{
    // 取五元组与时间戳低32位，使用密钥进行哈希
    u32 seq = secure_tcp_seq(ip_src, ip_dst, port_src, port_dst);
    // 将MSS索引编码到高位
    seq |= (tcp_mss_to_mss_idx(syn->mss) << 24) & 0xFF000000;
    return seq & 0x00FFFFFF | (tcp_time_stamp << 24); // 实际Linux实现有更复杂的位运算
}
```
注意：上述代码仅为演示原理，Linux内核的具体实现位于`net/ipv4/syncookies.c`，包含`tcp_v4_syn_recv_sock`等完整逻辑。

### 4. 常见误区与进阶思考
误区1：认为SYN Cookie只在遭受攻击时才启用。实际参数`net.ipv4.tcp_syncookies`有三个值（0关闭、1启用、2仅当SYN队列溢出时启用），Linux默认值为1（实际不同发行版可能为1或0），但语义是“当SYN队列（半连接队列）已满时自动启用”，并非总是启用。工程师常误解为配置为1系统就永远用SYN Cookie，从而影响对状态和性能的预期。

误区2：忽略SYN Cookie对TCP扩展选项的破坏。SYN Cookie机制在SYN-ACK中无法携带SACK、WScale（窗口缩放因子等）等选项，因为Cookie只编码了MSS，没有空间保存所有选项。这会导致经过SYN Cookie建立的连接在数据传输时出现性能下降（如窗口缩放未启用、无法使用SACK）。专业工程师在优化高并发TCP服务时，必须意识到被SYN Cookie保护而建立的连接可能与正常握手建立的连接在TCP选项上存在差异，从而影响吞吐。

思考题：假设攻击者能够预知服务端的SYN Cookie密钥（secret），在传统三次握手中，服务端收到ACK后，如何区分一个合法的第三次ACK与一个伪造的ACK？SYN Cookie方案中，服务端是否有办法防止重放攻击（即攻击者截获SYN-ACK后回放ACK）？如果密钥定期更换，如何处理客户端延迟了64秒以上的ACK？请从同步机制的本质出发，解释为什么即使攻击者伪造ACK，服务端仍然能拒绝非法连接，以及时间窗校验的粒度（Linux中为64秒）与系统时钟的关系。
