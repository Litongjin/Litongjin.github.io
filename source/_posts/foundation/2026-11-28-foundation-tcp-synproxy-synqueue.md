---
title: "每日基础技术总结 · 2026-11-28 · TCP SYN Flood攻击的Synproxy机制与半连接队列（syn_queue）溢出处理"
date: 2026-11-28 07:03:07
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-28 · TCP SYN Flood攻击的Synproxy机制与半连接队列（syn_queue）溢出处理

## 📚 今日主题

> **TCP SYN Flood攻击的Synproxy机制与半连接队列（syn_queue）溢出处理**（网络基础）

### 1. 核心概念速览
SYN Flood 的本质是攻击协议状态机的资源不对称：伪造源地址的 SYN 使服务端在内核中创建 request_sock、发送 SYN+ACK 并等待最终 ACK，半连接队列（syn_queue/request_sock_queue）被不可达的半开连接占满，导致合法 SYN 无法进入三次握手。Synproxy 是在内核数据平面（通常经 netfilter）对 SYN 做前置代理与源可达性验证的机制：先由代理使用合成序列号或 syncookie 与客户端完成首次握手，确认客户端可接收并回 ACK 后，再向后端建立或放行真实连接。半连接队列溢出处理则定义内核在 syn_queue 满时的策略：丢弃、启用 syncookies 无状态兜底，或由 SYNPROXY 隔离。它位于操作系统内核网络栈与防火墙数据平面之间，是连接层可用性、负载均衡、推理服务网关和分布式训练通信稳定性的底层基础；专业工程师必须掌握，否则无法区分应用层超时、内核队列耗尽与安全过滤策略。

### 2. 底层原理剖析
1. 监听套接字的两类队列：Linux listen socket 维护半连接队列 syn_queue（request_sock_queue，保存 SYN_RECV 的 request_sock）与全连接队列 accept_queue（保存已完成握手、等待 accept 的 sock）。前端常见的 Node HTTP server.listen(port, backlog) 或 HTTP 网关 backlog，主要映射到 accept_queue 上限；syn_queue 上限由 tcp_max_syn_backlog、listen backlog 与内存约束决定。
2. 正常被动打开流程：SYN 到达 -> 查找 listen sock -> 分配 request_sock -> 记录 MSS、SACK、窗口缩放等选项 -> 选择服务器 ISN -> 入 syn_queue -> 发 SYN+ACK -> 等待 ACK。ACK 到达 -> 用 ack_num 定位 request_sock -> 创建 full sock -> 从 syn_queue 移除 -> 入 accept_queue -> accept() 返回。
3. 溢出处理：当 syn_queue 无法继续入队时，若 tcp_syncookies=0，通常直接丢弃 SYN；若启用 syncookies，内核可发送基于密码学派生的 SYN+ACK，其 ISN 编码客户端四元组、MSS 与选项摘要，不占用 request_sock。客户端 ACK 携带 ISN+1，内核通过重算 cookie 验证，然后创建 full sock。syncookies 是无状态兜底，不是连接代理。
4. SYNPROXY 流程：客户端 SYN -> netfilter SYNPROXY 拦截 -> 以代理身份回 SYN+ACK（可用 cookie 或合成状态）-> 客户端 ACK -> 代理验证客户端真实可达 -> 代理再向后端发起 SYN 或将报文导入后端连接 -> 后续数据在代理、conntrack 或 NAT 路径中转发。后端 listen socket 在客户端完成验证前不会看到攻击 SYN，因此后端 syn_queue 不被伪造半开连接消耗。
5. 与前端已有知识对比：前端调用 fetch 或 axios 时，TCP connect 是网络栈黑盒；若服务端 syn_queue 溢出，客户端表现为 connect 超时或 SYN 无响应，应用层通常没有请求日志，因为 socket 尚未进入 accept。Node HTTP server 的 backlog 不是半连接队列；这与 TS 接口只在编译期约束类型不同，TCP listen、backlog、syn_queue 是运行时内核资源配额，直接决定系统调用和网卡报文路径的行为。

### 3. 基础代码与实战验证
```text
// 内核被动打开的精确伪代码：展示 syn_queue、syncookies、SYNPROXY 的分支
on_syn(listener, pkt):
  if synproxy_enabled(pkt):
      // SYNPROXY 在 netfilter 前置接管，不让未验证 SYN 到达 listener
      send_synack_with_proxy_cookie(pkt)
      return DROP_TO_STACK

  if syn_queue_full(listener):
      if tcp_syncookies_enabled():
          isn = hmac_derive(client_ip, client_port, server_ip, server_port, mss, opts)
          // cookie 藏在 ISN 中；不分配 request_sock，不占半连接队列
          send_synack(pkt, isn)
      else:
          drop(pkt) // 队列满且无 cookie，合法客户端也会超时
      return

  req = alloc_request_sock()
  req.isn = select_server_isn()
  req.client_opts = parse_tcp_options(pkt)
  if !syn_queue_enqueue(listener, req):
      // 入队失败仍可能进入 syncookies 兜底
      free(req)
      if tcp_syncookies_enabled():
          send_synack(pkt, derive_cookie_isn())
      else:
          drop(pkt)
      return
  send_synack(pkt, req.isn)

on_ack(listener, pkt):
  if synproxy_state(pkt):
      // 客户端已通过代理首次握手；此后代理与后端完成二次握手或做序列号转换
      proxy_establish_or_forward(pkt)
      return

  req = syn_queue_lookup(listener, ack = pkt.ack_num - 1)
  if req:
      child = create_full_sock(req)
      syn_queue_remove(req)
      accept_queue_enqueue(listener, child) // 等待 accept()
  else if tcp_syncookies_enabled() and cookie_valid(pkt):
      child = create_full_sock_from_cookie(pkt)
      accept_queue_enqueue(listener, child)
  else:
      drop(pkt)

# 主机侧验证半连接与全连接状态
ss -lnt 'sport = :8080'
# Recv-Q：accept_queue 当前长度；Send-Q：listen backlog 上限
ss -ant state syn-recv | wc -l
# 统计处于 SYN-RECV 的半连接；SYN Flood 时该值逼近半连接上限

# 实验参数（仅在隔离环境）
sysctl -w net.ipv4.tcp_max_syn_backlog=16
sysctl -w net.ipv4.tcp_syncookies=1
# backlog 参数影响 accept_queue，不等同于半连接队列
# 下面 C 片段用于观察全连接堆积，不能单独压满 syn_queue

#include <netinet/in.h>
#include <sys/socket.h>
#include <unistd.h>

int main(void) {
  int fd = socket(AF_INET, SOCK_STREAM, 0);
  struct sockaddr_in addr = {0};
  addr.sin_family = AF_INET;
  addr.sin_port = htons(8080);
  bind(fd, (struct sockaddr*)&addr, sizeof(addr));
  listen(fd, 1); // 主要限制 accept_queue，为 1 便于观察全连接溢出
  pause(); // 不调用 accept，让已完成握手连接滞留内核
}
```

### 4. 常见误区与进阶思考
误区一：把 listen(backlog)、somaxconn 或 Node HTTP backlog 当成半连接队列上限。它们主要约束 accept_queue；syn_queue 由 tcp_max_syn_backlog、内存与 syncookies/SYNPROXY 策略决定。
误区二：认为 syncookies 或 SYNPROXY 开启后 SYN Flood 完全消失。syncookies 只是避免半连接状态耗尽，仍要消耗 CPU、带宽与序列号验证成本；SYNPROXY 把半开状态转移到代理，并引入序列号转换、TCP 选项协商和连接跟踪开销，真实源慢连接或空连接仍需在应用层或连接层做超时与配额。
进阶思考：SYNPROXY 与客户端握手使用的 ISN 与后端真实服务器生成的 ISN 不同。若代理只简单转发报文而不重写 TCP 序列号和确认号，客户端与后端会在哪个报文开始失配？代理必须维护哪些状态，才能兼容 MSS、SACK、窗口缩放和时间戳选项？
