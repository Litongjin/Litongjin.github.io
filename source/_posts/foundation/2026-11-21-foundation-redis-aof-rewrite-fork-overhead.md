---
title: "每日基础技术总结 · 2026-11-21 · Redis持久化AOF重写期间的子进程fork开销分析"
date: 2026-11-21 07:01:55
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-21 · Redis持久化AOF重写期间的子进程fork开销分析

## 📚 今日主题

> **Redis持久化AOF重写期间的子进程fork开销分析**（后端基础）

### 1. 核心概念速览
核心概念速览：
Redis AOF 重写期间的 fork 开销，本质是内核执行 fork 时创建子进程并复制父进程虚拟地址空间元数据所引入的延迟、CPU 与后续写时复制缺页成本。Redis 在触发 bgrewriteaof 后由父进程调用 fork 创建子进程；子进程基于当前内存快照遍历键值并生成新 AOF；父进程在重写期间继续处理命令，并把写命令追加到 AOF rewrite buffer；子进程完成后，父进程将 buffer 刷入新 AOF 并替换旧文件。fork 的关键机制不是复制全部 Redis 数据，而是内核把可写匿名页标记为只读、复制页表与少量进程结构，子进程读共享物理页，父进程或子进程写共享页时触发缺页异常并按页复制。该知识点解决的是高吞吐内存数据库持久化过程中服务抖动、内存峰值、快照一致性与磁盘写入成本之间的权衡。它位于操作系统进程模型、虚拟内存、写时复制、日志持久化和数据库恢复的交汇点。前端工程师出身者尤其需要掌握，因为前端通常被运行时屏蔽掉 fork、页表、COW、mmap、fsync 等 POSIX 资源成本，而后端稳定性问题常来自内核级复制与 IO 控制。

### 2. 底层原理剖析
底层原理剖析：
机制：
1. Redis 单线程主循环发现需要重写 AOF，调用 aofStartRewriting 或直接处理 bgrewriteaof，准备重写缓冲区。
2. 父进程执行 pid = fork();。内核为子进程创建新的进程描述符，复制父进程文件描述符表、信号处理表、虚拟内存区域和页表；匿名可写物理页不立即复制，只改为共享只读并登记 COW 元数据。
3. 子进程关闭或隔离非必要文件描述符，打开临时 AOF 文件，遍历数据库键值，把可恢复命令序列写入临时 AOF；子进程的写主要落在自己新分配的堆页与文件描述符缓冲，不是重写整个 Redis 数据。
4. 父进程在 fork 返回后恢复事件循环，所有新写命令被 append 到 aofRewriteBuffer，同时 Redis 主键空间发生更新；如果更新命中 fork 后仍与子进程共享的页，内核触发缺页异常，复制该页后再写入。
5. 子进程写完并 fsync 临时文件后通知父进程；父进程将 aofRewriteBuffer 追加到临时 AOF，fsync 后 rename 替换正式 AOF，并释放缓冲区。

伪代码：
bgrewriteaof() {
  if (rewrite_in_progress) return;
  rewrite_buffer = new_list();
  pid = fork(); // 内核复制页表与进程元数据；匿名页通过 COW 共享
  if (pid == 0) {
    tmp = open(AOF_REWRITE_TMPFILE, O_CREAT | O_WRONLY | O_TRUNC);
    rewriteAppendAllKeysToAof(tmp); // 子进程从当前共享内存读 key/value 并生成命令
    fsync(tmp);
    reportRewriteSuccessToParent();
    _exit(0);
  } else {
    startParentListeningChildState();
    continue_serving_clients();
    while (rewrite_running) {
      command = process_client_command();
      appendToRewriteBufferIfWriteCommand(command); // 父进程写旧共享页时可能触发 COW
    }
  }
}
rewrite_completed() {
  fsync(tmp);
  appendRewriteBufferToTmpFile(); // 补写重写期间父进程产生的新命令，保证快照后增量
  rename(tmp, final_aof);
}

开销来源：
A. fork 系统调用阻塞父进程：Redis 主线程不可重入，内核复制页表期间不能处理命令；实例越大，页表项越多，复制和元数据管理越久。
B. COW 缺页：父进程写入共享页时，内核分配新物理页、复制旧页、更新页表；随机小写会让大量页分散触发缺页，表现为延迟长尾。
C. 内存增长：重写期间的 dirty 页复制、aofRewriteBuffer、子进程临时缓冲和父进程新分配都会推高 RSS；极端场景下接近或超过原内存两倍。
D. 磁盘与 CPU 竞争：子进程持续序列化命令并写盘，fsync 可阻塞子进程；但父进程卡顿的主要内核级变量是 fork 与 COW，而不是单纯磁盘带宽。
E. 大页与分配器：透明大页可减少页表项，却可能把一次写复制放大到 2MB；THP 页迁移、内存碎片整理和 NUMA 策略也会让 fork 后的写延迟更不可控。

与前端已有概念的异同：
1. Node.js child_process.fork 创建独立 V8 isolate 与 IPC，不共享 JS 堆，不存在 POSIX COW；Redis fork 与 Node fork 同名但语义完全不同：Redis fork 是内核共享物理页的低成本快照，Node fork 是进程隔离并复制运行环境。
2. Web Worker 与 SharedArrayBuffer 强调跨线程共享同一块 ArrayBuffer 并用原子变量同步；Redis AOF rewrite 用进程级只读页实现共享，写时复制不依赖显式锁，靠页保护触发内核处理。
3. 前端不可变数据或 persistent data structure 的 COW 发生在应用数据结构层，复制的是共享子树；Redis fork 的 COW 发生在虚拟内存层，复制的是物理页。前者影响 JS 对象复杂度，后者影响 RSS、TLB、页表与缺页异常。
4. 浏览器多进程模型隔离渲染，进程创建成本高；Redis fork 依赖 COW 避免复制堆，但把成本转移到页表复制和后续缺页复制，这是服务端长期稳定运行必须量化的内核成本。

### 3. 基础代码与实战验证
```text
验证代码：
import os
import mmap
import resource

# 分配 16 MB 匿名内存并触碰每一页，保证 fork 前已有可观察页表项
buf = mmap.mmap(-1, 16 * 1024 * 1024)
for i in range(0, 16 * 1024 * 1024, 4096):
    buf[i] = 0

r0 = resource.getrusage(resource.RUSAGE_SELF).ru_minflt

pid = os.fork()
if pid == 0:
    # 子进程只读取共享页，不触发写时复制；对应重写子进程遍历内存生成 AOF
    _ = buf[0]
    r1 = resource.getrusage(resource.RUSAGE_SELF).ru_minflt
    print('child read delta_minflt=%d' % (r1 - r0))
    os._exit(0)

# 父进程写入共享页时触发缺页异常并复制一个物理页；对应重写期间业务写命令
buf[0] = 1
r2 = resource.getrusage(resource.RUSAGE_SELF).ru_minflt
print('parent write delta_minflt=%d' % (r2 - r0))
os.waitpid(pid, 0)

结果判读：parent write delta 通常大于 child read delta；若继续逐页写入 16 MB，parent 缺页数会接近 16 MB / 4 KB，直接暴露 COW 按页复制成本。
```

### 4. 常见误区与进阶思考
常见误区：
1. 认为 fork 会立即复制整份 Redis 内存，所以 bgrewriteaof 的内存峰值一定固定为 2 倍。事实是 fork 初始只复制页表和内核元数据，物理页共享；峰值接近 2 倍取决于重写期间父进程对共享页的写入比例、dirty 页数量、aofRewriteBuffer 大小以及子进程临时内存。纯读流量或低频写入时，额外内存可能远小于 2 倍；高频随机写和预分配 heap 被写穿时，才逼近甚至超过 2 倍。
2. 认为 AOF 重写卡顿只与磁盘 IO 有关。事实是 Redis 主线程在 fork 内核调用期间被阻塞，无法处理事件；随后大量随机小写触发的 COW 缺页也会制造延迟长尾。磁盘带宽和 fsync 影响子进程写新 AOF 的速度，但解释不了秒级命令停顿；必须把 fork 时长、RSS 增量、dirty 页速率、缺页异常速率和命令延迟一起观测。

进阶思考：
在一个 64 GB 实例上开启透明大页并执行 AOF 重写，为什么父进程可能比 4 KB 页模式更平滑地 fork，却更容易出现重写期间的写延迟尖峰？需要从 PMD 数量减少、页表复制成本下降、2 MB 粒度 COW 复制、khugepaged 页迁移与内存碎片整理、NUMA 远端分配等机制给出量化解释。
