---
title: "每日基础技术总结 · 2026-06-12 · WebSocket 心跳与关闭帧"
date: 2026-06-12 08:00:00
categories: [技术分享]
tags: ["技术分享", "网络基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-06-12 · WebSocket 心跳与关闭帧

## 📚 今日主题

> **WebSocket 心跳与关闭帧**（网络基础）

### 1. 核心概念速览
WebSocket 心跳与关闭帧是WebSocket协议（RFC 6455）中用于维护连接活性与优雅终止连接的两个核心控制帧机制。心跳（通常为Ping/Pong帧）用于探测对端是否存活、检测网络半开连接，防止中间设备（如NAT、代理）因空闲超时回收连接；关闭帧（Close帧，opcode 0x8）用于在应用层主动或被动地发起关闭握手，确保双方都收到关闭意图并释放资源。它们在协议栈中位于应用层之下、TCP传输层之上，属于WebSocket协议自身的连接管理原语，不依赖上层业务逻辑。专业工程师必须掌握其底层机制，因为生产环境中的连接泄漏、假死连接、优雅下线、负载均衡踢除、分布式推送可靠性等问题，本质上都是对心跳与关闭帧语义理解不深导致的。

### 2. 底层原理剖析
WebSocket帧格式统一：FIN（1bit）+ RSV（3bit）+ opcode（4bit）+ MASK（1bit）+ Payload len（7/16/64bit）+ Masking-key（可选）+ Payload。控制帧（opcode 0x8关闭、0x9 Ping、0xA Pong）的Payload长度必须≤125字节，且控制帧不能被分片（FIN必须为1）。

心跳机制本质：Ping帧由任一端发送，携带可选的应用数据（≤125字节）；对端收到Ping后必须立即回复一个Pong帧，Pong的Payload必须与收到的Ping完全一致。协议并未规定心跳发送频率，但常规实现（如浏览器WebSocket API）不暴露Ping/Pong帧，而是由底层自动响应。服务端实现（如ws、Netty）需要自己维护心跳定时器，定期发送Ping，并设置超时检测Pong响应，若超时则认为连接已死，主动发送Close或直接关闭TCP。

关闭握手机制：任一端可发送Close帧，包含2字节的状态码（如1000正常关闭、1001 going away、1002协议错误、1003不支持数据类型、1009消息过大、1011内部错误、1012/1013服务重启等）和可选的UTF-8原因字符串。收到Close帧后，对端必须回送一个Close帧（如果尚未发送），然后关闭底层TCP连接。这是一个四次握手？实际上，协议规定：A发Close，B收到后应发送Close回应，然后B主动关闭TCP，A收到B的Close后也关闭TCP。因此，从应用层看是“两个Close帧”，但TCP层可能出现四次FIN。注意：发送Close后，不能立即关闭TCP，必须等待对端Close帧或超时（通常实现中关闭后立即销毁socket，但为了可靠，可设置超时等待）。

与前端已有概念的对比：前端工程师熟悉HTTP的请求-响应模型，HTTP是半双工、短连接（HTTP/1.1 keep-alive可复用，但仍是请求-响应），WebSocket是全双工长连接。HTTP没有心跳概念，因为每次请求都能探测活性；WebSocket长连接必须有心跳。WebSocket的Close帧类似于TCP的FIN，但更上层；同时类似于HTTP的Connection: close，但WebSocket关闭是一个显式的协议级协商过程，而HTTP关闭只是TCP层的FIN。若类比JS的事件循环，Ping/Pong像事件循环中的异步回调，但它是协议层的强制要求，不是应用层逻辑。

### 3. 基础代码与实战验证
```text
以下使用Node.js的net模块实现一个极简的WebSocket帧解析与心跳/关闭处理（不依赖第三方库），仅展示底层原理。

const net = require('net');

// 解析WebSocket帧：返回 { opcode, fin, payload }，这里仅处理控制帧
function parseFrame(buffer) {
  const firstByte = buffer[0];
  const opcode = firstByte & 0x0F;
  const fin = (firstByte & 0x80) !== 0;
  const secondByte = buffer[1];
  const masked = (secondByte & 0x80) !== 0;
  let payloadLength = secondByte & 0x7F;
  let offset = 2;
  if (payloadLength === 126) {
    payloadLength = buffer.readUInt16BE(2);
    offset += 2;
  } else if (payloadLength === 127) {
    payloadLength = Number(buffer.readBigUInt64BE(2));
    offset += 8;
  }
  const maskKey = masked ? buffer.slice(offset, offset + 4) : null;
  offset += masked ? 4 : 0;
  const payload = masked ? Buffer.from(buffer.slice(offset, offset + payloadLength).map((byte, i) => byte ^ maskKey[i % 4])) : buffer.slice(offset, offset + payloadLength);
  return { opcode, fin, payload };
}

// 构造一个WebSocket帧（这里仅构造控制帧，不掩码，因为服务端→客户端帧不要求掩码）
function buildFrame(opcode, payload = Buffer.alloc(0)) {
  const header = Buffer.alloc(2);
  header[0] = 0x80 | opcode; // FIN=1，控制帧必须FIN
  header[1] = payload.length; // ≤125
  return Buffer.concat([header, payload]);
}

const server = net.createServer((socket) => {
  console.log('客户端连接');
  let buffer = Buffer.alloc(0);
  let heartbeatTimer = null;

  socket.on('data', (data) => {
    buffer = Buffer.concat([buffer, data]);
    // 简化：假设一个数据包只包含一个完整帧，实际需要循环解析
    while (buffer.length >= 2) {
      const frame = parseFrame(buffer);
      if (frame.opcode === 0x8) { // 收到关闭帧
        console.log('收到Close帧，状态码', frame.payload.readUInt16BE(0));
        // 回复Close帧（状态码1000）
        const closeReply = Buffer.alloc(2);
        closeReply.writeUInt16BE(1000, 0);
        socket.write(buildFrame(0x8, closeReply));
        socket.end(); // 关闭TCP连接
        clearInterval(heartbeatTimer);
        break;
      } else if (frame.opcode === 0x9) { // 收到Ping，必须立即回复Pong
        console.log('收到Ping，Payload:', frame.payload.toString());
        socket.write(buildFrame(0xA, frame.payload)); // Pong回显相同Payload
        buffer = buffer.slice(2 + (frame.payload.length)); // 简化长度
      } else if (frame.opcode === 0xA) { // 收到Pong，说明对端存活
        console.log('收到Pong，心跳存活确认');
        buffer = buffer.slice(2 + (frame.payload.length));
      } else {
        // 其他帧忽略，仅演示控制帧
        buffer = Buffer.alloc(0);
      }
    }
  });

  // 服务端主动心跳：每10秒发送Ping，若5秒内没有Pong则关闭
  heartbeatTimer = setInterval(() => {
    console.log('发送Ping');
    socket.write(buildFrame(0x9, Buffer.from('heartbeat')));
    let pongReceived = false;
    const pongListener = (data) => {
      // 实际需要解析，这里简化为收到任何数据即视为存活
      pongReceived = true;
      socket.removeListener('data', pongListener);
    };
    socket.once('data', pongListener);
    setTimeout(() => {
      if (!pongReceived) {
        console.log('Pong超时，关闭连接');
        socket.destroy();
      }
    }, 5000);
  }, 10000);

  socket.on('close', () => {
    console.log('连接关闭');
    clearInterval(heartbeatTimer);
  });
});

server.listen(8080);

注释说明：buildFrame中opcode 0x9是Ping，0xA是Pong，0x8是Close。控制帧必须FIN=1。服务端不掩码，客户端必须掩码。parseFrame处理了掩码解码。真实场景需处理粘包/半包，这里为了清晰只解析单帧。
```

### 4. 常见误区与进阶思考
误区1：认为收到Close帧后可以立即销毁TCP连接而不必回送Close帧。实际上，协议要求收到Close后必须回送一个Close帧，否则对端可能因为未完成关闭握手而一直等待或产生RST，导致应用层无法区分是正常关闭还是异常中断。正确做法：收到Close后，回送Close（携带相同或合理状态码），然后关闭TCP（FIN）。

误区2：将心跳与业务超时混为一谈。WebSocket的Ping/Pong是协议层的活性探测，与业务层的请求超时无关。有些工程师在应用层自己实现心跳（如发送业务消息作为心跳），这会导致中间设备无法识别WebSocket控制帧，且会污染业务数据。必须使用协议规定的Ping/Pong帧。另外，浏览器端WebSocket API不暴露发送Ping的能力，只能收到Pong（实际上浏览器会自动响应Ping），所以服务端必须主动发Ping，不能依赖客户端。

思考题：如果服务端发送Ping后，客户端一直不回复Pong，但TCP连接仍然处于ESTABLISHED状态（没有FIN或RST），服务端应该依靠什么机制来判定连接死亡？请从TCP的keep-alive、WebSocket的心跳超时、以及应用层业务消息三个层次分别阐述，并说明为什么仅仅依赖WebSocket的心跳超时在分布式系统中可能仍然不够（例如，客户端机器睡眠后恢复，TCP连接可能存活，但客户端进程已死）。实际上，TCP keep-alive默认2小时不活跃才探测，WebSocket Ping/Pong可自定义，但客户端进程假死时可能不响应Pong，因此服务端需设定一个合理的心跳超时阈值（如30秒），超过则主动发送Close并关闭TCP。更底层的问题是：TCP的FIN只能在优雅关闭时收到，进程崩溃或网络分区不会触发FIN，因此心跳是唯一可靠的活性信号。
