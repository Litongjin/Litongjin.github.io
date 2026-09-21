---
title: "每日基础技术总结 · 2025-05-27 · Redis 主从复制中的部分重同步（psync）"
date: 2025-05-27 20:00:00
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2025-05-27 · Redis 主从复制中的部分重同步（psync）

## 📚 今日主题

> **Redis 主从复制中的部分重同步（psync）**（后端基础）

### 1. 核心概念速览
部分重同步（Partial Resynchronization，简称 PSync）是 Redis 主从复制协议中的一种增量同步机制，旨在解决主节点故障切换或网络瞬断后的数据一致性恢复问题。其本质是通过维护主节点的复制偏移量（Repl Offset）和主节点的运行 ID（Run ID），在 Slave 检测到 Master 变更时，尝试使用上一次记录的 Run ID 和最后接收的 Replication Offset 进行握手。如果 Master 能证明其数据集仍包含 Slave 缺失的部分数据片段（即在内存缓存中保留了足够的历史命令缓冲区），则仅需传输缺失的命令流而非全量快照。该机制在高可用架构（如 Redis Sentinel 或 Cluster）中至关重要，它大幅降低了因主节点故障导致的 RTO（恢复时间目标），避免了全量复制带来的高 I/O 开销和网络带宽消耗，保障了分布式存储系统的平滑容灾能力。

### 2. 底层原理剖析
PSync 的运行逻辑严格依赖于两个核心标识：1. Master Run ID：唯一标识一个特定的 Master 实例进程；2. Master Replication Offset：表示 Master 已发送给 Slave 的数据总量字节数。

交互流程如下：
1. 当 Slave 与 Master 断开连接后重连，或在接收到新的 SYNC 命令时，Slave 若曾建立过完整同步，则发起 PSYNC <run_id> <offset> 请求。
2. Master 验证逻辑分为两种路径：
   - 路径 A（完全重同步）：如果 Run ID 不匹配（说明 Master 发生了选举更换，旧进程退出新进程启动），或者 Offset 对应的缓冲数据已过期（Full Sync Buffer 已满或已被覆盖），Master 响应 ERR 并触发 BGSAVE，执行全量同步（PSYNC <run_id> <offset> -> FULLRESYNC <new_run_id> <new_offset>）。这等同于重新执行第一次完整的 SYNC 流程。
   - 路径 B（部分重同步）：如果 Run ID 匹配（同一 Master 进程重启），且 Offset 差值落在当前保留的复制积压缓冲区（Replication Backlog）范围内，Master 仅发送该时间段内的新增命令。这需要 Master 在主线程执行写操作前，将写命令同时推入回退日志（Backlog）中。

对比前端概念：这与 TypeScript 中的‘结构类型系统’有本质不同，Redis PSync 更像是一种严格的‘契约校验’。Run ID 如同类的构造函数签名，一旦改变即代表实体已重构；Offset 如同时间戳或版本号。只有当‘身份’未变且‘状态差异’在服务端可追溯范围内时，才允许增量更新。否则，必须重置状态（全量同步），以防止数据脏读或不一致。这不同于前端 React 的虚拟 DOM Diff，后者是局部最优解的比对，而 PSync 是基于中心化权威数据源的全局一致性保证。

### 3. 基础代码与实战验证
```text
// Redis C 语言核心逻辑伪代码演示 (redis.c)
void replicationCron() {
    // 定期检查主从连接状态
    if (!master->repl_transfer) { 
        // 如果处于传输中断状态，尝试重连
        if (server.master_replid[0] != ' ') {
            // 构造 PSYNC 请求
            char *cmd = createClientPsync(server.master_replid, 
                                         server.master_repl_offset);
            sendToMaster(cmd); 
            // 注意：这里并未立即等待结果，而是由事件循环处理回调
        }
    }
}

// Master 侧处理逻辑 (sds cstring psync_buf)
int processCommand(client *c) {
    if (strcasecmp(c->argv[0]->ptr, "psync") == 0) {
        sds requested_runid = c->argv[2]->ptr;
        long long requested_offset = strtoll(c->argv[3]->ptr, NULL, 10);
        
        // 关键判断 1: Run ID 是否变更？(主节点是否发生脑裂后的新选举)
        int can_partial = (strncasecmp(server.master_replid, 
                                       requested_runid, SERVER_REPLID_SIZE) == 0);
        
        if (can_partial) {
            // 关键判断 2: Offset 是否在 backlog 有效范围内？
            // server.repl_backlog_histlen 记录当前积压缓冲区的有效长度
            // server.repl_backlog_off 记录缓冲区的起始偏移量
            if (requested_offset >= (server.repl_backlog_off - server.repl_backlog_histlen)) {
                // 触发部分重同步：只写入 backlog 中对应区间的命令
                writeToReplicas(&server.slaves, requested_offset); 
                return...;
            } else {
                // 缓冲区已失效，强制降级为全量同步
                callFullResync(...);
            }
        } else {
            // Run ID 不一致，主节点角色已变更，必须全量同步
            callFullResync(...);
        }
    }
}
```

### 4. 常见误区与进阶思考
['误区一：认为部分重同步可以无限期保留历史记录。实际上，主节点的复制积压缓冲区（Replication Backlog）大小是有限的（由 repl-backlog-size 配置，默认 1MB）。一旦写入的数据量超过此限制，旧的偏移量将不可追溯，此时即使 Run ID 不变，也会被迫触发全量同步。专业工程师需根据业务写入吞吐量合理调整该参数，避免频繁 Full Sync 造成的性能抖动。', '误区二：混淆网络断连与主节点宕机。单纯的 TCP 断连（无 Master 重启）通常能利用 Keep-Alive 快速恢复或触发极短时间的重连逻辑，但若伴随 Master 进程 Restart，必须经过 Run ID 校验。很多初级开发者误以为只要 IP 不变就能直接增量同步，忽略了 Redis 内部通过 Run ID 防止‘僵尸脑裂’副本进入集群的设计意图。']
