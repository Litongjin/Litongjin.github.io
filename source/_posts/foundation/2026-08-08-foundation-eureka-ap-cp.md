---
title: "每日基础技术总结 · 2026-08-08 · Eureka 的服务发现与自保护模式：AP 与 CP 的取舍"
date: 2026-08-08 08:00:00
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-08-08 · Eureka 的服务发现与自保护模式：AP 与 CP 的取舍

## 📚 今日主题

> **Eureka 的服务发现与自保护模式：AP 与 CP 的取舍**（后端基础）

### 1. 核心概念速览
Eureka 是 Netflix 开源的服务注册中心，本质是一个基于 REST 的、点对点（Peer-to-Peer）复制模式的服务发现系统。服务实例启动时向 Eureka Server 注册自身元数据（IP、端口、健康状态、租约信息），消费者通过拉取（全量/增量）或本地缓存读取注册表以定位目标服务地址。Eureka 在 CAP 三角中明确选择 AP：优先保证可用性（Availability）与分区容错性（Partition Tolerance），牺牲强一致性（Consistency）。其核心机制包含两条主线：(1) 租约机制——客户端每 30s 发送心跳续约，租约默认有效期 90s，服务端后台线程周期性扫描并剔除过期租约；(2) 自我保护模式——当最近 15 分钟内实际续约数低于期望续约数的 85%（renewalPercentThreshold），Eureka 判定可能发生了网络分区（而非实例故障），于是停止剔除任何过期实例，宁可让注册表包含部分失效数据，也不清空注册表导致服务雪崩。该模式是分布式系统中『故障检测』不确定性的直接体现：网络分区时，服务端无法区分『实例已死』与『实例活着但心跳不可达』。掌握 Eureka 的 AP 取舍，是理解所有分布式注册中心（ZooKeeper/etcd 的 CP 路线、Nacos 的双模式）以及分布式系统 CAP 理论落地的关键入口，也是后端工程师设计高可用微服务基础设施的必修课。

### 2. 底层原理剖析
Eureka 的底层运行机制可拆解为三个子机制：

一、租约心跳与续约
每个实例注册后持有租约（Lease），记录 lastHeartbeatTimestamp。客户端每 30s（默认）发送一次心跳（REST: PUT /eureka/apps/{appId}/{instanceId}），服务端更新租约时间戳。后台 EvictionTask 每 60s 执行一次租约扫描，剔除条件为：now - lastHeartbeatTimestamp > leaseDuration（默认 90s）。

二、期望续约数与自我保护阈值
服务端维护两个关键计数器：
expectedNumberOfRenewsPerMin = 当前注册实例数 × (60s / 客户端心跳间隔)
numberOfRenewsPerMinThreshold = expectedNumberOfRenewsPerMin × renewalPercentThreshold（默认 0.85）
每分钟末，服务端统计实际续约数 numberOfRenewsPerMin，若 numberOfRenewsPerMin < numberOfRenewsPerMinThreshold，则进入自我保护模式：EvictionTask 的 isLeaseExpirationEnabled() 返回 false，所有过期租约不再被剔除。退出条件：实际续约数恢复至阈值以上，且持续一段时间，自动退出。

三、点对点复制与最终一致
Eureka Server 节点间无主从之分，每个节点既接受读写，也通过 PeerReplication 将注册变更（注册、续约、下线）异步复制给其他节点。节点间传播存在延迟，因此任意时刻不同节点看到的注册表可能不一致，属于最终一致性模型。这与 ZooKeeper 的 ZAB 协议（Leader 写、多数派确认、强一致）形成鲜明对比。

与前端已有知识体系的对比：
- Eureka 的最终一致性 ≈ 前端状态管理中的『乐观更新』哲学：UI 先展示结果（可用性优先），后端异步确认后再修正（最终一致）。若后端失败则回滚，但用户已获得响应。
- Eureka 的自我保护 ≈ 前端的『降级策略』：当检测到外部依赖不稳定时，优先保证主流程可用，返回可能过期的缓存数据（stale-while-revalidate），而不是直接报错。
- Eureka Server 节点对等 ≈ Web 前端的 CDN 边缘节点：无中心、各自缓存、异步回源同步，任何节点可独立服务。
- 对比 Java 接口与 TS 接口的差异（类型系统层面）：Java 接口是编译期类型契约 + 运行时多态分派，TS 接口是编译期结构类型检查（structural typing），运行时不存在。这与 Eureka 的 AP/CP 取舍同属一个思维模式：不同层面对『一致性』的定义和实现强度不同——Java 接口保证的是编译期强一致，TS 接口是编译期结构兼容，而 Eureka 在运行期主动放弃强一致换取可用性。

核心伪代码（EvictionTask 判定逻辑）：

    // 每60秒执行
    void evict() {
        if (isLeaseExpirationEnabled()) {           // 自我保护未触发或已恢复
            for (lease : registry) {
                if (now - lease.lastHeartbeat > leaseDuration) {
                    registry.remove(lease.instanceId);   // 正常剔除
                }
            }
        }
        // 自我保护模式下：不执行任何剔除，保留过期实例
    }

    boolean isLeaseExpirationEnabled() {
        if (!selfPreservationEnabled) return true;        // 手动关闭自我保护
        return actualRenewsPerMin > renewsPerMinThreshold; // 实际续约数高于阈值才允许剔除
    }

### 3. 基础代码与实战验证
```text
以下为极简 Java 模拟程序，复现 Eureka 的核心：租约心跳、期望续约数阈值计算、自我保护触发与剔除跳过。不依赖任何框架，直接运行 main 即可观察行为。

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class EurekaSelfPreservationDemo {
    // 租约：记录实例最近心跳时间
    static class Lease {
        final String instanceId;
        volatile long lastHeartbeatMs;
        Lease(String id) { this.instanceId = id; this.lastHeartbeatMs = System.currentTimeMillis(); }
    }

    // 配置常量（与 Eureka 默认值对齐）
    static final long LEASE_DURATION_MS = 90_000;         // 租约有效期 90s
    static final long RENEW_INTERVAL_MS = 30_000;         // 客户端心跳间隔 30s
    static final double RENEWAL_PERCENT_THRESHOLD = 0.85; // 自我保护触发阈值

    final Map<String, Lease> registry = new ConcurrentHashMap<>();
    volatile boolean selfPreservationMode = false;        // 自我保护模式标志
    volatile int expectedRenewsPerMin = 0;                // 期望续约数/分钟 = 实例数 × (60/30)
    volatile int actualRenewsLastMin = 0;                 // 上一分钟实际收到的续约数

    // 客户端心跳续约：服务端仅更新租约时间戳并累加实际续约数
    public void renew(String instanceId) {
        Lease lease = registry.get(instanceId);
        if (lease != null) {
            lease.lastHeartbeatMs = System.currentTimeMillis();
            actualRenewsLastMin++;
        }
    }

    // 模拟每分钟一次的 EvictionTask
    public void evictionTaskTick() {
        // 1. 重新计算期望续约数（实例数 × 每分钟心跳次数）
        expectedRenewsPerMin = registry.size() * (int) (60_000 / RENEW_INTERVAL_MS);

        // 2. 判断是否进入自我保护：实际续约数 < 期望值 × 0.85
        int threshold = (int) (expectedRenewsPerMin * RENEWAL_PERCENT_THRESHOLD);
        if (expectedRenewsPerMin > 0 && actualRenewsLastMin < threshold) {
            selfPreservationMode = true;
            System.out.println("[TRIGGER] 实际续约=" + actualRenewsLastMin
                + " 阈值=" + threshold + " -> 进入自我保护模式，跳过剔除");
        } else if (actualRenewsLastMin >= threshold) {
            // 3. 实际续约恢复后自动退出自我保护
            selfPreservationMode = false;
            System.out.println("[RECOVER] 实际续约=" + actualRenewsLastMin
                + " 阈值=" + threshold + " -> 退出自我保护模式");
        }

        // 4. 遍历租约执行剔除（仅在非自我保护模式下）
        long now = System.currentTimeMillis();
        for (Lease lease : registry.values()) {
            boolean expired = now - lease.lastHeartbeatMs > LEASE_DURATION_MS;
            if (expired && !selfPreservationMode) {
                registry.remove(lease.instanceId);  // 正常模式：剔除过期实例
                System.out.println("[EVICT] 剔除过期实例 " + lease.instanceId);
            } else if (expired && selfPreservationMode) {
                System.out.println("[HOLD] 自我保护中，保留过期实例 " + lease.instanceId);
            }
        }
        actualRenewsLastMin = 0;  // 重置每分钟计数器
    }

    public static void main(String[] args) throws InterruptedException {
        EurekaSelfPreservationDemo demo = new EurekaSelfPreservationDemo();
        // 注册 3 个实例
        for (int i = 1; i <= 3; i++) {
            demo.registry.put("instance-" + i, new Lease("instance-" + i));
        }

        // 第 1 分钟：全部实例正常心跳 -> 实际续约数 = 3×2 = 6，阈值 = 6×0.85 = 5，不触发
        for (String id : demo.registry.keySet()) { demo.renew(id); demo.renew(id); }
        demo.evictionTaskTick();

        // 第 2 分钟：只有 1 个实例心跳（其余疑似网络分区） -> 实际续约数 = 2，阈值 = 5，触发自我保护
        demo.renew("instance-1"); demo.renew("instance-1");
        demo.evictionTaskTick();

        // 第 3 分钟：网络分区恢复，全部实例心跳 -> 实际续约数 = 6，退出自我保护
        for (String id : demo.registry.keySet()) { demo.renew(id); demo.renew(id); }
        demo.evictionTaskTick();
    }
}

预期输出：
第 1 分钟：正常，无剔除。
第 2 分钟：输出 [TRIGGER] 进入自我保护模式，instance-2/instance-3 租约已过期但被 [HOLD] 保留。
第 3 分钟：输出 [RECOVER] 退出自我保护模式。

核心注释：renew() 中的 actualRenewsLastMin++ 对应 Eureka 的 RenewProcessor 对每分钟续约计数器的累加；evictionTaskTick() 中的 threshold 计算对应 AbstractInstanceRegistry.updateRenewsPerMinThreshold()；selfPreservationMode 对应 isLeaseExpirationEnabled() 的返回值控制剔除开关。
```

### 4. 常见误区与进阶思考
误区一：『自我保护模式是故障，应该关闭它』
这是最危险的认知。自我保护是 Eureka 应对网络分区（而非实例崩溃）的核心防御机制。网络分区时，大量健康实例的心跳无法到达服务端，若此时正常执行剔除逻辑，注册表会被瞬间清空，所有消费者拿到空注册表，引发全链路雪崩。关闭自我保护（eureka.server.enable-self-preservation=false）等于放弃对网络分区故障的隔离能力，仅适合单机或完全受控内网环境。正确理解：自我保护牺牲的是『注册表的准确性』，换取的是『注册中心的可用性』——宁可返回过期的实例列表，也不返回空列表。

误区二：『Eureka 注册表是强一致的，注册后立即可被消费者发现』
实际上，Eureka 的注册信息要经过三层延迟才能到达消费者：(1) 服务端节点间的异步 Peer 复制（无主从，最终一致）；(2) 消费者本地缓存（默认 30s 拉取一次全量注册表）；(3) 自我保护模式下过期实例不被剔除，消费者可能持续拿到已失效的地址。这与前端开发中『服务端状态 ≠ 客户端状态』的认知同构：Redux 中的 store 是客户端的『缓存』，必须自己处理过期与失效，不能假设与服务端实时同步。

进阶思考题：
假设一个微服务集群中，实例 A 所在主机突然宕机（非网络分区），恰在此时 Eureka 因另一区域的网络抖动触发了自我保护模式。A 的租约在 90s 后过期，但由于自我保护，A 不会被剔除，消费者通过 Eureka 获取到的注册表中始终包含 A。请问：消费者调用 A 时会发生什么？消费者如何感知 A 已不可用？如果 A 是集群中最后一个可用实例（其余实例此前已因故障被剔除），消费者的行为又是什么？请结合消费者侧的负载均衡重试机制（如 Ribbon 的 retry 策略）与 Eureka 的客户端缓存刷新机制，描述完整的故障链路与最终恢复路径。
