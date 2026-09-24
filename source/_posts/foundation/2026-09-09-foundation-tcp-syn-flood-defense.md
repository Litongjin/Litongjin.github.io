---
title: "每日基础技术总结 · 2026-09-09 · TCP的SYN Flood防御与半连接队列满时的丢弃策略"
date: 2026-09-09 08:00:00
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-09 · TCP的SYN Flood防御与半连接队列满时的丢弃策略

## 📚 今日主题

> **TCP的SYN Flood防御与半连接队列满时的丢弃策略**（后端基础）

### 1. 核心概念速览
TCP SYN Flood 是一种利用 TCP 三次握手协议的拒绝服务攻击：攻击者发送大量伪造源地址的 SYN 包，使服务器在 SYN 队列（半连接队列）中残留大量未完成连接，最终队列耗尽，合法连接无法建立。防御的本质不是阻止 SYN 到达（通常无法阻止），而是控制 SYN 队列溢出后的行为，确保系统在恶意负载下仍能服务合法请求。核心机制包括 SYN Cookie、SYN Proxy、动态队列调整和选择性丢弃。该知识点位于网络协议栈与操作系统内核资源管理的交界处，是 DDoS 防护、负载均衡和高并发服务设计的底层细节。专业工程师必须掌握，因为仅靠应用层重试或水平扩展无法应对协议栈级别的资源耗尽，理解它有助于深入理解 TCP 状态机、内核网络栈及连接转发技术。

### 2. 底层原理剖析
TCP 握手在内核中维护两个队列：半连接队列（SYN Queue）保存已收到 SYN 但未完成 ACK 的请求；全连接队列（Accept Queue）保存已完成握手等待 accept() 的连接。SYN Flood 的本质是让 SYN Queue 充满。

内核处理收到 SYN 的典型逻辑：
· 先在监听 socket 的哈希表中查找对应四元组的 request_sock（半连接条目）。
· 若不存在，则是新握手尝试，尝试分配新的 request_sock；若队列长度已满，则丢弃该 SYN。
· 若存在，说明是重传 SYN，则刷新定时器，不会再次入队。

当 SYN 队列满时，防御机制的策略如下：

1. 直接丢弃：最简单，但攻击者不断发送新 SYN，队列永远满，合法请求被饿死。
2. SYN Cookie：不分配 request_sock，将握手的必要状态（MSS、时间戳等）编码进初始序号（ISN）。收到客户端 ACK 时，通过序列号逆运算重建连接。代价是丢失大量 TCP 选项（如窗口缩放、SACK、时间戳），且解码校验需要 CPU 开销。
3. SYN Proxy：由中间设备（如 LVS、防火墙）代为完成握手，成功后与后端再握手，避免后端直接接触恶意 SYN。
4. 调优参数：增大 tcp_max_syn_backlog 只能延长填满时间；tcp_abort_on_overflow=1 会在 accept 队列满时直接发送 RST，迫使客户端重连，而不是静默丢弃。

伪代码逻辑：

on SYN:
  if tcp_syncookies && 队列可能满:
    if 四元组已存在: update_timer
    else: send_syn_cookie()  // 不分配队列条目
  else:
    if syn_queue_len >= max_backlog:
      if 四元组已存在: update_timer
      else: drop_packet()
    else:
      allocate_request_sock()
      enqueue_to_syn_queue()

与前端体系的对比：类似浏览器事件队列中的背压机制——当宏任务队列过长，新任务被丢弃或延迟，但已有的 pending 任务仍继续执行。区别在于 TCP 队列是内核资源受限的数据结构，状态转换严格，并且面对的是对抗性恶意流量而非普通负载；前端背压通常由应用层控制，TCP 的 SYN 丢弃发生在内核态，应用层毫无感知。

### 3. 基础代码与实战验证
```text
# 以下命令在 Linux 上验证 SYN Flood 防御与队列行为，需 root 权限。

# 查看当前半连接队列长度（SYN-RECV 状态的连接数）
ss -t state syn-recv -n | wc -l

# 开启 SYN Cookie，开启后新 SYN 不再占用半连接队列条目
sysctl -w net.ipv4.tcp_syncookies=1

# 查看当前最大半连接队列长度
cat /proc/sys/net/ipv4/tcp_max_syn_backlog

# 将最大半连接队列人为调小（例如 64），快速制造溢出条件
sysctl -w net.ipv4.tcp_max_syn_backlog=64

# 当全连接队列（accept 队列）满时的行为：1=发送 RST，0=静默丢弃 SYN
sysctl -w net.ipv4.tcp_abort_on_overflow=0

# ---- 最小验证模型 ----
# 启动一个 Python 监听 socket，listen(1) 但永不 accept，使全连接队列满

python3 - <<'EOF'
import socket, time
s = socket.socket()
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
s.bind(('0.0.0.0', 9999))
s.listen(1)          # backlog=1，全连接队列容量为 1
while True:          # 永不调用 accept()，队列保持满
    time.sleep(1)
EOF

# 客户端连接：第一个连接成功完成握手进入 accept 队列，第二个连接也会进入队列但满？
# 实际监听队列，backlog=1 表示队列最多放 1 个已完成握手连接，第三个 SYN 到来时触发溢出策略。
# 验证点：观察第三个连接是否被 RST（tcp_abort_on_overflow=1）还是超时重传（=0）。
```

### 4. 常见误区与进阶思考
误区 1：把 SYN Cookie 当作万能的防御。SYN Cookie 只能缓解半连接队列耗尽，无法防御带宽耗尽或全连接队列攻击（如 HTTP Flood）；并且启用后 TCP 选项（窗口缩放、SACK、时间戳）会被压缩或丢失，导致高吞吐长连接性能显著下降。

误区 2：误以为调大 tcp_max_syn_backlog 可以防 SYN Flood。该参数只是扩大了资源池，攻击者只需发送更多 SYN 照样填满，同时每个半连接条目会占用约 1KB 内核内存，盲目调大可能因内存耗尽而宕机。正确姿势是结合 SYN Cookie、SYN Proxy、IP 白名单和业务层限流。

思考题：当半连接队列已满时，一个已存在半连接的四元组重传 SYN 与一个全新的四元组 SYN 在内核查找 request_sock 哈希表的路径有何不同？为什么后者会被丢弃而前者能刷新定时器？请从 TCP 的时间戳机制和 request_sock 哈希查找角度解释。
