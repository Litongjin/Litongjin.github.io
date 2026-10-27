---
title: "每日基础技术总结 · 2026-10-27 · TCP Timestamps选项在测量RTT与保护PAWS序列号回绕中的作用"
date: 2026-10-27 11:29:41
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-27 · TCP Timestamps选项在测量RTT与保护PAWS序列号回绕中的作用

## 📚 今日主题

> **TCP Timestamps选项在测量RTT与保护PAWS序列号回绕中的作用**（网络基础）

### 1. 核心概念速览
### 1. 核心概念速览
TCP Timestamps 是 TCP 选项（Kind=8，Length=10，RFC 7323/1323），携带两个 32-bit 字段：TSval 与 TSecr。启用前提：SYN/SYN-ACK 双方均携带该选项，协商成功后每个数据/ACK 段可携带；TSval 为发送端维护的单调 timestamp clock，TSecr 回显最近收到的对端 TSval。本质：在 32-bit 序列号空间之外引入一条每连接、单调递增的 32-bit 时间轴，用于两个目标：RTTM（RTT Measurement）在含重传场景下测量 RTT；PAWS（Protect Against Wrapped Sequence numbers）拒绝旧序列号回绕后仍可能匹配的陈旧段。它位于传输层 TCP 状态机内部，是高速长延迟网络（高 BDP）中正确性、RTT 估计与拥塞控制的基础设施。专业工程师必须掌握：否则无法解释 Karn 算法、重传 RTT 二义性、序列号回绕、PAWS 丢包、时间戳选项协商失败，以及现代 QUIC 为何改用不会回绕的 packet number 等设计取舍。

### 2. 底层原理剖析
### 2. 底层原理剖析
TSopt 线格式：Kind=8 | Length=10 | TSval(4B) | TSecr(4B)。TSval 由发送端本地 timestamp clock 生成，只要求单调递增，不要求与对端同步，RFC 建议频率 1ms~1s。TSecr 在 ACK 中回显“最近接受的、按序到达的”对端 TSval。接收端维护两个状态：ts_recent = 最近接受的 TSval；ts_recent_age = 更新 ts_recent 的本地时刻。

RTTM：发送段 S 时记录 TSval=T1。对端收到后，在后续 ACK 的 TSecr 中回显 T1。发送端收到 ACK 后计算 RTT = now_local - TSecr。因为 ACK 回显的是具体发送段的 TSval，重传时若重传段带新 TSval=T2，ACK 回显 T2 即测量重传 RTT；若回显 T1 则识别为原段 ACK。这消除了 Karn 算法在重传时无法区分 ACK 属于原发送还是重传的二义性，使 RTT/拥塞窗口在丢包恢复期仍可更新。

PAWS：TCP 序列号仅 32-bit，在高 BDP 链路上会快速回绕。仅用 seq 无法区分“新段”与“回绕前的旧段”。PAWS 在接收端对每个带 TSopt 的段执行：若 TSval 比 ts_recent 旧，且 ts_recent_age < 24 天，则丢弃；否则接受并更新 ts_recent=TSval、ts_recent_age=now。24 天阈值用于老化 ts_recent，避免长期停滞或连接复用导致误丢新段。32-bit TSval 比较不是普通无符号比较，而是模 2^32 的有符号语义：若 (ts_recent - TSval) mod 2^32 < 2^31，则 TSval 旧。这样即使 TSval 自身回绕，仍能判断新旧。

与前端已有概念对比：performance.now() 是单调时钟，Date.now() 可被系统时间回拨；TCP TSval 类似“每连接私有 performance.now()”，但不共享 epoch，不能用于跨主机绝对时间。前端常用递增 request id/version 防止旧响应覆盖新状态；PAWS 在传输层用 32-bit TSval 扩展 32-bit seq 的顺序空间。TypeScript interface 是编译期结构类型契约，TCP Timestamps 是运行时二进制 TLV 能力协商：未在 SYN 阶段双方同时声明，后续 TSval/TSecr 语义不存在。

### 3. 基础代码与实战验证
```text
### 3. 基础代码与实战验证
以下 Python 片段仅用标准库 struct/time 模拟 TSopt 解析、PAWS 判定与 RTTM 计算。

import struct, time

KIND_TS = 8
MASK32 = 0xffffffff
HALF32 = 0x80000000
TS_RECENT_MAX_AGE_MS = 24 * 24 * 60 * 60 * 1000  # RFC 老化阈值：24 天

def parse_tsopt(buf):
    kind, length = buf[0], buf[1]
    assert kind == KIND_TS and length == 10
    tsval, tsecr = struct.unpack('!II', buf[2:10])
    return tsval, tsecr

def paws_old(tsval, ts_recent):
    # 模 2^32 有符号比较：若 ts_recent - tsval 小于 2^31，则 tsval 在 ts_recent 之前，即旧
    return ((ts_recent - tsval) & MASK32) < HALF32

class TCPConn:
    def __init__(self):
        self.ts_enabled = False
        self.ts_recent = 0
        self.ts_recent_age = 0
        self.rtt_ms = None

    def now_ms(self):
        # 本地单调时钟，仅保证不回退；不要求与对端同步
        return int(time.monotonic() * 1000) & MASK32

    def build_tsopt(self, tsecr):
        tsval = self.now_ms()
        # 发送时写入本端 TSval，并回显对端最近 TSval 到 TSecr
        return struct.pack('!BBII', KIND_TS, 10, tsval, tsecr)

    def on_segment(self, tsval, tsecr, has_tsopt=True):
        if not (self.ts_enabled and has_tsopt):
            return 'ACCEPT_NO_TS'
        now = self.now_ms()
        # PAWS：旧 TSval 且 ts_recent 未老化，丢弃，以防序列号回绕后的陈旧段被接受
        if paws_old(tsval, self.ts_recent) and ((now - self.ts_recent_age) & MASK32) < TS_RECENT_MAX_AGE_MS:
            return 'DROP_BY_PAWS'
        if not paws_old(tsval, self.ts_recent):
            self.ts_recent = tsval
            self.ts_recent_age = now
        # RTTM：ACK 的 TSecr 回显本端曾经发出的 TSval，故 RTT = 本端当前时钟 - TSecr
        if tsecr != 0:
            self.rtt_ms = (now - tsecr) & MASK32
        return 'ACCEPT'

验证要点：构造 ts_recent=1000，收到 tsval=900 时 paws_old 为 True，若 age 未超 24 天则丢包；收到 tsval=1001 时更新 ts_recent。构造本端发出 TSval=5000，当前 now_ms=5030，ACK 中 TSecr=5000，则 rtt_ms=30。重传时若重传段使用新 TSval，ACK 回显新值即可测重传 RTT。
```

### 4. 常见误区与进阶思考
### 4. 常见误区与进阶思考
误区一：把 TSval 当作墙上时钟或要求两端同步。TSval 只是发送端本地单调计数器，单位由实现决定，两端 epoch 无关；RTT 计算使用的是本端当前时钟减去 ACK 回显的 TSecr，而不是对端时间。若误以为需要 NTP 同步，会错误解释 RTTM 和 PAWS。

误区二：认为 PAWS 彻底解决序列号回绕。PAWS 只提供额外的 32-bit 单调时间维度，仍受 TSval 自身回绕、时钟不单调、24 天老化、选项协商失败、中间件剥离选项等影响；它不能替代对窗口、BDP、MSL 和实现过滤逻辑的正确理解。另外，RTTM 并非每个 ACK 都产生无偏样本：合并 ACK、乱序、SACK 可能使 TSecr 对应非被确认段，实现需过滤或保守使用。

思考题：在 100Gbps 链路上，32-bit TCP 序列号按字节计数的回绕周期约为 2^32*8/10^11 ≈ 0.34 秒；若 RTT=200ms，BDP≈2.5GB，超过 2^32 字节的一半。为什么 RFC 建议 timestamp clock 频率为 1ms~1s，而不是每包递增？请从 TSval 回绕周期、ts_recent 24 天老化、PAWS 模 2^32 比较和 MSL 的关系推导：PAWS 究竟是在什么时间尺度上保护什么，以及若 TSval 频率过高会引入什么失效模式。
