---
title: "每日基础技术总结 · 2024-05-11 · 架构取舍：单体 vs 微服务 vs Serverless"
date: 2024-05-11 20:00:00
categories: [技术分享]
tags: ["技术分享", "分布式与架构设计"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-05-11 · 架构取舍：单体 vs 微服务 vs Serverless

## 📚 今日主题

> **架构取舍：单体 vs 微服务 vs Serverless**（分布式与架构设计）

### 1. 核心概念速览
1. 核心概念速览
架构模式本质是资源隔离、通信机制与部署粒度的组合策略。

- 单体架构 (Monolith)：所有业务逻辑编译打包为单一进程（如 JVM/Hercules 实例）。共享内存空间，同步调用，无网络序列化开销。解决的是开发复杂度低、部署简单的确定性执行问题。在 AI 体系中，对应早期单节点模型训练或推理。
- 微服务架构 (Microservices)：将系统拆分为多个独立部署的进程/容器，通过轻量级 HTTP/gRPC 协议进行异步或同步通信。每个服务拥有独立数据库。解决的是大规模协作下的团队自治、独立扩容与异构技术栈集成问题。引入了分布式系统的复杂性（CAP 定理权衡、最终一致性）。
- Serverless (FaaS + BaaS)：基于事件驱动的函数即服务。开发者仅关注业务代码逻辑，基础设施由云厂商完全抽象。按请求时长计费，具备冷启动延迟特性。解决的是峰值流量下的极致弹性伸缩与运维成本最低化问题。适用于无状态、短生命周期任务。

专业工程师必须掌握，因为架构决策直接决定了系统的吞吐量上限、容错边界、数据一致性模型及总拥有成本 (TCO)。

### 2. 底层原理剖析
2. 底层原理剖析

核心差异在于进程管理权与控制权的让渡程度：

- 控制粒度：单体在应用层控制线程调度；微服务在 OS/Container 层利用 IPC 或 Network Namespace 隔离；Serverless 在 Hypervisor/K8s Pod 层由调度器动态分配 Kernel 资源。
- 通信模型：单体是 Process-local Call (零拷贝)；微服务是 RPC over TCP/IP (序列化/反序列化、上下文传递)；Serverless 是 Event Bus/Webhook (消息队列解耦 + 自动扩缩容控制器)。
- 状态管理：单体依赖本地堆栈或共享 DB；微服务强制无状态设计以支持横向扩展；Serverless 要求外部化状态 (Ephemeral State)。

前端对比：前端 SPA 中 React/Vue 组件是 DOM/Diff Algorithm 层面的‘微服务’，但运行在同一 JS 主线程（单体进程）。Serverless 类似前端调用的 API Gateway + Lambda，前端只需关注触发条件 (Event)，无需关心后端 Node.js 实例的生命周期管理 (Cold Start vs Hot Instance)。

### 3. 基础代码与实战验证
```text
3. 基础代码与实战验证

// Node.js 极简演示：三种模式的内核差异

// [单体] 直接内存引用，同步执行，无网络 IO 延迟
function monolithModule() {
    const data = { id: 1, value: 'A' };
    // 内部方法直接访问闭包变量，零序列化开销
    return processBusiness(data); 
}

// [微服务] 模拟 HTTP RPC 调用，涉及序列化与网络传输
const http = require('http');
async function microserviceCall(url) {
    return new Promise((resolve) => {
        // 关键机制：HTTP 协议头 + JSON Body 序列化
        const req = http.get(url, (res) => {
            let body = '';
            res.on('data', chunk => body += chunk);
            res.on('end', () => resolve(JSON.parse(body)));
        });
    });
}

// [Serverless] 事件驱动入口，依赖外部环境注入
// 真实环境由 CloudWatch/Kinesis 触发，此处模拟 Event Context
async function serverlessHandler(event, context) {
    // 关键机制：幂等性处理（因冷启动可能复用容器也可能新建）
    const input = event.body ? JSON.parse(event.body) : event;
    const result = await externalDatabaseQuery(input.id);
    // 返回标准化响应，不包含任何基础设施信息
    return { statusCode: 200, body: JSON.stringify(result) };
}
```

### 4. 常见误区与进阶思考
4. 常见误区与进阶思考

- 认知误区 1：认为微服务一定优于单体。忽略分布式事务（2PC/TCC）和链路追踪带来的巨大运维熵增。对于中小规模团队，单体+模块化目录结构往往具备更高的交付效率。
- 认知误区 2：认为 Serverless 是终极形态。忽略冷启动延迟（Cold Start）对实时交互场景的影响，以及供应商锁定（Vendor Lock-in）导致的不可迁移性。

进阶思考题：
在设计一个高并发秒杀系统时，如果选择 Serverless 架构，如何从底层机制上解决‘雪崩效应’？请结合云厂商的并发限制（Concurrency Limits）与队列削峰原理，推导其背后的资源隔离策略。
