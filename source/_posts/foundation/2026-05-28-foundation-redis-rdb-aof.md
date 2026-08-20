---
title: "每日基础技术总结 · 2026-05-28 · Redis 的 RDB 与 AOF 持久化"
date: 2026-05-28 08:00:00
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-05-28 · Redis 的 RDB 与 AOF 持久化

## 📚 今日主题

> **Redis 的 RDB 与 AOF 持久化**（后端基础）

### 1. 核心概念速览
RDB（Redis Database）与 AOF（Append Only File）是 Redis 提供的两种持久化机制，用于将内存中的数据集保存到磁盘，以应对进程退出或服务器宕机后的数据恢复。RDB 本质是某一时刻内存数据的全量二进制序列化快照，由 fork 出的子进程完成写入，利用操作系统写时复制（COW）保证父进程服务不中断；AOF 本质是记录 Redis 服务器收到的每个写命令的追加日志，按配置策略（always/everysec/no）将命令同步到磁盘，恢复时通过重放命令重建数据集。二者解决的核心问题是：在非零数据丢失容忍度下，如何平衡数据安全、恢复速度与运行时性能。该机制处于存储引擎与操作系统文件系统之间的抽象层，是理解 Redis 高可用、备份恢复、混合持久化以及分布式一致性基础（如复制、哨兵）的前提。专业工程师必须掌握其底层原理，才能在容量规划、故障恢复、性能调优中做出正确决策，而不是停留在使用 save/appendonly 命令的表面。

### 2. 底层原理剖析
RDB 机制：
1. 触发条件：手动 SAVE（同步阻塞）或 BGSAVE（异步 fork）；配置自动触发如 save 900 1（900 秒内至少 1 次写操作）。
2. BGSAVE 流程：父进程调用 fork() 创建子进程，子进程继承父进程的完整内存页表，但物理内存页共享。子进程开始将内存数据序列化写入临时 RDB 文件，期间父进程继续处理写请求，若某页被修改，触发 COW，父进程复制该页后修改，子进程看到的仍是旧页，从而保证快照的一致性。
3. 完成后临时文件原子重命名为 dump.rdb，替换旧文件。

AOF 机制：
1. 每个写命令（如 SET、LPUSH）以 Redis 协议文本格式追加到 aof_buf 缓冲区，再根据 appendfsync 配置决定何时调用 fsync 刷盘：always（每条命令同步刷盘，最安全但性能最差）、everysec（每秒刷盘，最多丢 1 秒数据）、no（由操作系统决定刷盘时机，性能最好但可能丢更多数据）。
2. 恢复时逐条读取命令并重放，重建整个数据集。
3. 由于命令不断累积，AOF 文件会膨胀，需要 AOF 重写（bgrewriteaof）：fork 子进程将当前内存数据转换为恢复所需的最小命令集，生成新 AOF 文件并原子替换，同时缓冲重写期间的新命令以保证不丢失。

与前端概念的对比：RDB 类似于前端状态管理中的“全量序列化快照”——例如将整个 Redux store 序列化到 localStorage，恢复时整体反序列化；AOF 则类似于“事件溯源/操作日志”——记录每次 action，恢复时重新派发所有 action 以重建状态。本质差异在于：RDB 保存的是“状态”（结果），恢复快但丢失两次快照间的全部修改；AOF 保存的是“操作”（过程），恢复慢但可按策略控制丢失粒度。另外，Redis 的 fork/COW 机制是前端环境（浏览器）不具备的系统级原语，这也是为何前端持久化通常只能同步阻塞且数据量受限。

### 3. 基础代码与实战验证
```text
以下为验证 RDB 与 AOF 核心行为的 Redis 命令序列（通过 redis-cli 执行），每行注释说明底层运作机制。

# 查看当前持久化配置
CONFIG GET save
CONFIG GET appendonly

# 开启 AOF（动态生效，无需重启）
CONFIG SET appendonly yes
# 设置 AOF 刷盘策略为 everysec（每秒 fsync）
CONFIG SET appendfsync everysec

# 写入数据
SET user:1 "alice"
INCR page:views

# 触发 RDB 快照（后台 fork 子进程写临时文件）
BGSAVE
# 检查 RDB 是否成功
LASTSAVE

# 触发 AOF 重写（fork 子进程压缩命令日志）
BGREWRITEAOF

# 模拟重启后恢复：先关闭 Redis，再启动，观察数据是否还在
SHUTDOWN NOSAVE
# 启动后验证
GET user:1

# 验证 RDB 与 AOF 同时存在时，加载优先级（AOF 优先）
# 在启动日志中可看到 "DB loaded from append only file" 或 "DB loaded from disk"

# 若想观察 COW 效果，可在 BGSAVE 期间持续写入大量数据，并用 INFO stats 观察 rdb_saves 和 fork 耗时
```

### 4. 常见误区与进阶思考
误区 1：认为 AOF 一定比 RDB 可靠。实际上，AOF 的可靠性完全取决于 appendfsync 策略：always 才可能做到最多丢一条命令，everysec 可能丢一秒，no 则可能丢大量数据。而 RDB 虽然每两次快照之间数据全丢，但快照文件本身是原子生成的，不会出现半写状态。混合持久化（RDB 作为基线 + AOF 记录增量）才是现代 Redis 兼顾恢复速度与数据安全的手段。
误区 2：认为 RDB 的 BGSAVE 对父进程性能无影响。fork 本身需要复制页表，内存越大 fork 耗时越长；COW 在写多场景下会复制大量内存页，导致物理内存暴涨，甚至触发 OOM。因此大内存实例需要谨慎配置自动 save 策略。

思考题：当 Redis 同时开启 RDB 和 AOF 时，启动时加载哪个文件？为什么这样设计？请从数据完整性与文件解析成本两个角度分析。
