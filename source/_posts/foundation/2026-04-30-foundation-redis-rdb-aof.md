---
title: "每日基础技术总结 · 2026-04-30 · Redis 持久化：RDB 与 AOF 取舍"
date: 2026-04-30 20:00:00
categories: [技术分享]
tags: ["技术分享", "数据库与缓存进阶"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-04-30 · Redis 持久化：RDB 与 AOF 取舍

## 📚 今日主题

> **Redis 持久化：RDB 与 AOF 取舍**（数据库与缓存进阶）

### 1. 核心概念速览
RDB (Redis Database) 与 AOF (Append Only File) 是 Redis 实现数据持久化的两种核心机制，本质解决的是‘内存数据在进程崩溃或重启后的状态恢复’问题。RDB 是通过 fork 子进程将内存快照序列化到磁盘文件的机制，侧重于数据一致性（Consistency）和备份效率；AOF 是通过追加写入每条写命令到日志文件的机制，侧重于数据完整性（Integrity）和实时性。在计算机体系中，这是 Volatile Storage (RAM) 向 Non-Volatile Storage (Disk) 落盘的 I/O 优化策略。专业工程师必须掌握，因为在高并发、分布式及 AI 向量数据库等场景下，存储引擎的选型直接决定了系统对宕机容忍度（MTTR）和数据一致性的 SLA。

principals": "1. RDB 原理：触发条件（Save/BGSAVE 定时或手动）。核心步骤：主进程调用 fork() 创建子进程 -> 子进程使用 Copy-on-Write (COW) 技术读取父进程内存页并序列化为二进制格式写入临时文件 -> 完成后原子替换原 RDB 文件。优点：文件紧凑，恢复速度快。缺点：fork 瞬间可能阻塞主线程，且两次快照间的数据可能丢失。\n2. AOF 原理：所有写命令追加至缓冲区 -> 根据 fsync 策略（always/everysec/no）决定是否刷盘。核心配置：appendfsync。优点：数据损失极少（最多损失最后一次 fsync 周期内的数据）。缺点：文件体积庞大，恢复速度慢于 RDB。\n3. 对比前端概念：RDB 类似 Git 的 Commit 快照（时间点一致，增量小但可能有延迟）；AOF 类似 Linux Kernel 的 writeback 缓存 + Journaling（逐条记录，保证事务顺序）。前端中，RDB 类似于 IndexedDB 的 transaction commit，AOF 类似于 WebSocket 推送的逐条操作记录。

code": "// 伪代码逻辑展示 RDB Fork 与 AOF Append 的核心流程\n\n// RDB 生成过程 (简化伪代码)\npid = fork(); // 操作系统级复制内存页表，物理内存暂未复制\nif (pid == 0) {\n    // 子进程执行\n    for page in memory_pages: \n        if page.is_dirty: continue; // COW 机制：若被修改则复制\n        serialize(page, buffer);\n    write_buffer_to_temp_file(buffer);\n    atomic_rename(temp_file, rdb_filename); // 原子替换\n    exit(0);\n}\nelse {\n    // 主进程继续处理请求，不阻塞（除非负载极高导致 fork 慢）\n}
\n// AOF 重写与刷盘策略示意\nvoid handle_write_cmd(Command cmd) {\n    append_to_aof_buffer(cmd); // 加入 OS Page Cache\n    if (should_fsync_now()) {   // 基于 appendfsync: always/ everysec\n        syscall fsync(file_fd); // 强制落盘，同步阻塞等待磁盘 IO\n    }\n}

pitfalls": "误区一：认为开启 AOF 就绝对安全。实际上，若未配置正确的 fsync 策略（如设为 no），数据可能在断电时大量丢失。误区二：混淆 RDB/AOF 混合模式下的加载优先级。Redis 启动时优先加载 AOF，因为 AOF 通常包含更完整的数据集合，RDB 仅用于辅助修复或加速启动前的部分状态恢复。思考题：在一个高写入吞吐量的场景下，如果同时开启 RDB (bgsave 1min) 和 AOF (everysec)，当主进程在两个机制触发的间隙崩溃，Redis 重启后最终数据是哪个机制的状态？为什么？（提示：考虑 Redis 启动时的合并策略及 fsync 的滞后性。）"
