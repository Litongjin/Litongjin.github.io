---
title: "每日基础技术总结 · 2024-06-06 · Spring Cloud：服务注册与发现（Nacos/Eureka）"
date: 2024-06-06 20:00:00
categories: [技术分享]
tags: ["技术分享", "Java 后端与 Spring 生态"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-06-06 · Spring Cloud：服务注册与发现（Nacos/Eureka）

## 📚 今日主题

> **Spring Cloud：服务注册与发现（Nacos/Eureka）**（Java 后端与 Spring 生态）

### 1. 核心概念速览
服务注册与发现是分布式系统中解决动态服务实例寻址问题的核心机制。其本质是将服务实例的网络位置（IP+Port）及其元数据抽象为命名空间中的资源，并通过中央注册中心作为单一事实来源（Single Source of Truth），协调生产者与消费者之间的解耦。在计算机体系结构中，它位于应用层之上，屏蔽了底层基础设施的波动性（如容器重启、扩缩容导致的 IP 变化），为微服务架构提供了可观测性和弹性伸缩的基础设施。专业工程师必须掌握它，因为它是构建高可用、高性能后端系统的基石，理解其内部的心跳保活、一致性协议和数据同步机制，是排查线上漂移、网络分区及脑裂问题的前提。

### 2. 底层原理剖析
运行机制基于发布-订阅模型与定期轮询/长轮询结合：1. 启动阶段：服务提供者向注册中心发起 HTTP 或 gRPC 请求，写入包含实例信息（Instance ID, IP, Port, Metadata）的 JSON/XML 结构体；2. 心跳阶段：客户端维持 Timer，定时发送 Health Check 信号（默认每30秒一次），若超时未收到ACK或连续N次失败，注册中心标记实例为DOWN并从服务列表中剔除；3. 发现阶段：消费者查询特定服务名（Service Name）对应的实例列表，支持本地缓存以减少对注册中心的直接依赖，实现最终一致性。与前端 TypeScript 接口的对比：TS 接口编译期检查静态类型定义，确保代码结构正确；服务注册表运行时管理动态状态，确保网络连通性正确。前者是静态契约，后者是动态路由表。Eureka 采用 AP 模型（可用性优先，保证集群内部分区时仍可读写），Nacos 支持切换 CP（ZAB 协议）或 AP，通过 Raft 选主保证强一致性场景下的数据可靠性。

### 3. 基础代码与实战验证
```text
// 伪代码展示 Nacos Client 的核心注册逻辑
public void registerInstance(String serviceName, String groupName, Instance instance) throws NacosException {
    // 1. 构造 PUT 请求，序列化实例对象
    String params = buildParams(instance);
    String body = buildBody(instance);
    
    // 2. 发送 HTTP PUT 到 Nacos Server (默认端口 8848)
    // 底层通过 OkHttp/JDK URLConnection 建立 TCP 连接
    HttpResult result = HttpClient.httpPut(serverAddr + Constants.CONFIG_CONTROLLER_PREFIX, null, headers, params); // 实际路径可能不同
    
    // 3. 服务端处理逻辑 (伪代码):
    // a. 校验参数合法性
    // b. 获取 Namespace -> Group -> Service 的三元组锁
    // c. 更新内存中的存储服务实例 Map<ServiceKey, List<Instance>>
    // d. 将变更事件推送到订阅者队列 (Push/Long-Poll)
    
    // 4. 开启本地心跳检测线程
    heartbeatExecutor.scheduleAtFixedRate(() -> {
        if (!isHealthy(instance)) {
            deregisterInstance(); // 非正常下线
        } else {
            sendHeartbeat();      // 续命
        }
    }, HEART_BEAT_INTERVAL, HEART_BEAT_INTERVAL, TimeUnit.MILLISECONDS);
}
```

### 4. 常见误区与进阶思考
常见误区一：混淆 '服务治理' 与 '负载均衡'。注册中心负责‘在哪里’（寻址），负载均衡器（如 Spring Cloud LoadBalancer 或 Sidecar）负责‘找哪一个’（选择策略）。误区二：忽视健康检查的配置陷阱。默认仅检测进程存活，若应用死锁但进程仍在运行，注册中心仍视为 UP，导致流量打入不可用实例。进阶思考题：在 Nacos 使用 Raft 协议实现 CP 模式时，如果集群中超过半数节点宕机，剩余节点是否还能接受新的服务注册请求？请结合 Paxos/Raft 的 Quorum 机制解释其行为及对业务的影响。
