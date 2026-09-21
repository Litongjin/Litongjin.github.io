---
title: "每日基础技术总结 · 2024-03-29 · TCP SYN Flood 与 SYN Cookie 机制"
date: 2024-03-29 20:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-03-29 · TCP SYN Flood 与 SYN Cookie 机制

## 📚 今日主题

> **TCP SYN Flood 与 SYN Cookie 机制**（网络基础）

### 1. 核心概念速览
TCP SYN Flood 是一种利用 TCP 三次握手状态机缺陷的资源耗尽型 DoS 攻击。攻击者发送大量伪造源 IP 的 SYN 报文，使服务器维持大量半连接（Half-Open Connection）状态，耗尽内核连接队列内存。SYN Cookie 是内核级防御机制，其本质是将连接请求中的客户端参数序列化为加密校验码（Cookie），替代传统的半连接状态存储。服务器不分配 PCB（Protocol Control Block）资源，仅在收到 ACK 时验证 Cookie 合法性。若合法则直接计算出初始序列号并恢复连接状态；若非法或无法重建状态，则丢弃数据包。该机制位于网络协议栈传输层，是保障分布式系统可用性、支撑高并发后端及 AI 推理服务稳定性的基础防线，专业工程师需掌握以理解负载均衡、反向代理及内核调优对业务稳定性的底层影响。

### 2. 底层原理剖析
传统三次握手中，Server 收到 SYN 后进入 SYN_RCVD 状态，分配内核结构体存放 client_ip, client_port, server_port, seq 等状态。SYN Cookie 机制下，Server 收到 SYN 时：1. 提取客户端 IP, Port 和当前时间戳 T；2. 计算哈希 H = HMAC(MD5/SHA)(secret_key, {T, client_ip, client_port})；3. 将 T 编码进 MSS (Maximum Segment Size) 字段或初始化序列号 ISN 的低 24 位；4. 构造 SYN+ACK 响应，ISN = (T << 24) | H；5. 立即释放临时资源。Client 回复 ACK 时，Server 重新计算预期 ISN'；若 ISN == ISN'，则从 Cookie 中解码出 T，检查 T 是否过期（通常 TTL 为 1-2 分钟）。若未过期且 Client Window > 0，则认为有效，动态生成 PCB 并继续握手。与前端概念类比：这类似前端 SPA 中用 JWT Token 替代 Session Server 存储用户状态。Session 依赖服务端内存（易被填满），Token 将状态序列化在客户端携带的服务端可验证签名中（无状态扩展性）。TS 接口定义静态类型约束，而 SYN Cookie 是一种动态的、基于密码学签名的运行时契约校验。

### 3. 基础代码与实战验证
```text
// Linux 内核视角下的简化逻辑伪代码
class TCPSynHandler {
    // 1. 接收 SYN 包
    onSynReceived(packet) {
        const clientIP = packet.srcIP;
        const clientPort = packet.srcPort;
        const timestamp = getCurrentTime();
        
        // 核心：不创建 PCB (socket 结构体)，而是计算 Cookie
        // cookie_value 包含了时间戳戳记和客户端信息的 MAC 签名
        const cookieValue = calculateSynCookie(
            secretKey, 
            timestamp, 
            clientIP, 
            clientPort,
            packet.mss 
        );
        
        // 将 Timestamp 嵌入 ISN 高字节，Cookie 嵌入低字节
        const isn = (timestamp << 24) | cookieValue;
        
        // 发送 SYN+ACK，此时服务器无需维护任何状态
        sendSynAck(clientIP, clientPort, isn);
    }

    // 2. 接收 ACK 包
    onAckReceived(packet) {
        const receivedIsn = packet.seq - 1; // TCP SEQ 减一为确认值
        const extractedTimestamp = receiveIsn >> 24;
        const extractedCookie = receiveIsn & 0xFFFFFF;

        // 验证时间戳是否过期（防止重放攻击和资源久存）
        if (isExpired(extractedTimestamp, SERVER_LIFETIME)) {
            dropPacket();
            return;
        }

        // 重新计算期望的 Cookie
        const expectedCookie = calculateSynCookie(
            secretKey, 
            extractedTimestamp, 
            packet.srcIP, 
            packet.srcPort,
            decodeMSSFromPacket(packet) 
        );

        // 比对校验，成功则重建连接状态
        if (expectedCookie === extractedCookie && packet.window > 0) {
            allocatePCB(); // 此时才真正分配内核资源
            completeHandshake();
        } else {
            sendRst(); // 拒绝无效连接
        }
    }
}
```

### 4. 常见误区与进阶思考
1. 性能误区：认为 SYN Cookie 会显著增加 CPU 负担。实际上，现代硬件支持 SIMD 指令加速 CRC/HMAC 计算，且相比于因内存耗尽导致的上下文切换开销和网络拥塞，CPU 开销可忽略不计。只有在极端恶意洪泛且 Hash 碰撞率高时才需注意。
2. 兼容性误区：认为 SYN Cookie 破坏了 TCP 选项协商。事实上，SYN Cookie 通过将 MSS 等信息编码进 ISN 或专用 Cookie 字段，并在 ACK 阶段还原，完美保留了 MSS 协商能力。但需注意，部分旧式防火墙或 NAT 设备可能修改 SYN 包中的可选字段，导致校验失败。

深度思考题：如果攻击者不仅伪造源 IP，还持续监听目标服务器的 SYN+ACK 响应，并能准确模拟第三次握手的 ACK 包（即已知 ISN），SYN Cookie 机制为何依然能保护服务器免受资源耗尽攻击？请结合‘状态生成时机’和‘资源分配触发点’进行原理解析。
