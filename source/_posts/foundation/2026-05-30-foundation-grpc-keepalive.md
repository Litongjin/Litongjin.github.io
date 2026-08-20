---
title: "每日基础技术总结 · 2026-05-30 · gRPC 流式调用与 keepalive"
date: 2026-05-30 08:00:00
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-05-30 · gRPC 流式调用与 keepalive

## 📚 今日主题

> **gRPC 流式调用与 keepalive**（后端基础）

### 1. 核心概念速览
gRPC 流式调用是基于 HTTP/2 长连接上的双向字节流通信模型，包括服务端流（Server streaming）、客户端流（Client streaming）、双向流（Bidirectional streaming）三种模式，加上一元调用（Unary）共四种。本质是 HTTP/2 Stream 上持续传输 DATA 帧序列，每条消息对应一个 DATA 帧，帧携带流标识和序列号。keepalive 是 gRPC 连接层的心跳机制，通过发送 HTTP/2 PING 帧探测对端存活及网络连通性，PING 帧不承载业务数据，接收端必须回 ACK。解决长连接空闲被中间设备断开、对端崩溃未感知等问题。在整个体系中，gRPC 位于应用层与传输层之间，流式调用是 RPC 调用与流式传输的组合，keepalive 是连接生命周期管理的基础设施。专业工程师必须掌握，因为流式调用是事件驱动架构、实时通信的基石，keepalive 是分布式系统稳定性的防线。

### 2. 底层原理剖析
HTTP/2 连接上可复用多个流。每个流有唯一 ID，客户端发起奇数 ID，服务端偶数 ID。帧类型包括 HEADERS（请求头，含 gRPC 路径）、DATA（消息体，5 字节前缀压缩标志 + 消息长度 + protobuf 字节）、PING（keepalive）、RST_STREAM（终止流）等。gRPC 流式调用时，客户端发送 HEADERS 帧开启流，然后持续发送 DATA 帧；服务端同样。双向流则双方同时发送。keepalive 由 HTTP/2 层实现，gRPC 通过 channel 参数配置，在无活动帧时按间隔发送 PING，对端在超时内未响应则触发连接断开。注意：PING 不参与流量控制，有最高优先级。
与前端已有概念的对比：HTTP/1.x 的 fetch 是请求-响应模式，无法多路复用；WebSocket 是独立协议，帧类型不同；gRPC 流式调用在 HTTP/2 上，与 Fetch API 的 stream reader 类似，但语义为 RPC。前端中的 async generator 可对应服务端流式调用的消费者，每次 next() 对应一个 DATA 帧。但 gRPC 的消息边界通过长度前缀和压缩标志来解析，与 WebSocket 的帧格式不同。
伪代码描述流创建：
发送端：
1. 构造 HTTP/2 HEADERS 帧，包含 :method=POST, :path=/package.Service/Method, content-type=application/grpc
2. 发送 DATA 帧，每个 DATA 帧负载前 5 字节为 1 字节 compressed-flag + 4 字节消息长度，随后是 protobuf 消息
3. 结束：发送 END_STREAM 标志。
keepalive 流程：
1. 定时器到期，若距离上次接收帧超过阈值，则发送 PING 帧
2. 对端收到 PING，立即发送 PING ACK
3. 若超时未收到 ACK，则判定连接不可用，关闭连接。

### 3. 基础代码与实战验证
```text
// server.js
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const packageDefinition = protoLoader.loadSync('hello.proto');
const hello = grpc.loadPackageDefinition(packageDefinition).hello;

const server = new grpc.Server();
server.addService(hello.Greeter.service, {
  SayHello: (call) => {
    for (let i = 0; i < 5; i++) {
      call.write({ message: 'Hello ' + call.request.name + ' #' + i }); // 每个 write 发送一个 DATA 帧
    }
    call.end(); // 发送 END_STREAM 标志，结束流
  }
});
server.bindAsync('0.0.0.0:50051', grpc.ServerCredentials.createInsecure(), () => {
  server.start();
});

// client.js
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const packageDefinition = protoLoader.loadSync('hello.proto');
const hello = grpc.loadPackageDefinition(packageDefinition).hello;

const client = new hello.Greeter('localhost:50051', grpc.credentials.createInsecure(), {
  'grpc.keepalive_time_ms': 10000,   // 每 10 秒发送 HTTP/2 PING 帧
  'grpc.keepalive_timeout_ms': 5000, // 5 秒未收到 ACK 则断开连接
  'grpc.keepalive_permit_without_calls': 1 // 无活动 RPC 时也允许 PING
});
const call = client.SayHello({ name: 'world' }); // 发起流式调用，返回 ClientReadableStream
call.on('data', (reply) => console.log('收到:', reply.message)); // 每个 DATA 帧触发
call.on('end', () => console.log('流结束'));

// hello.proto
syntax = 'proto3';
package hello;
service Greeter {
  rpc SayHello (HelloRequest) returns (stream HelloReply) {}
}
message HelloRequest { string name = 1; }
message HelloReply { string message = 1; }
```

### 4. 常见误区与进阶思考
常见误区：
1. 将 keepalive 理解为应用层心跳，试图在业务代码中发送自定义消息保活。实际上 keepalive 由 HTTP/2 PING 帧实现，由传输层自动发送，应用层无法拦截。若对端未响应，gRPC 库直接断开连接，业务层收到错误。配置后应避免再设计应用层心跳，以免重复消耗。
2. 混淆 gRPC 双向流式与 WebSocket。gRPC 双向流是 HTTP/2 流上的 RPC 调用，使用 protobuf 编码，依赖 HTTP/2 的流复用和流量控制；WebSocket 是独立协议，通过 HTTP Upgrade 建立，帧格式和应用模型不同。gRPC 流式无法直接在 WebSocket 上传输。
思考题：
如果客户端同时打开多个流，每个流分别传输不同消息，HTTP/2 如何确保各流的消息按序到达且不互相干扰？请从帧结构、流标识、流量控制三个角度解释。
