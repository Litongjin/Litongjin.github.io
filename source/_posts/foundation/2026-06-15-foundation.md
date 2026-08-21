---
title: "每日基础技术总结 · 2026-06-15 · 页缓存与脏页回写机制"
date: 2026-06-15 08:00:00
categories: [技术分享]
tags: ["技术分享", "操作系统基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-06-15 · 页缓存与脏页回写机制

## 📚 今日主题

> **页缓存与脏页回写机制**（操作系统基础）

### 1. 核心概念速览
页缓存（Page Cache）是内核为文件系统提供的一层透明缓存，以物理内存页为单位保存磁盘文件的数据副本。其本质是通过内存空间换磁盘 I/O 时间，利用局部性原理消除每次访问都穿透磁盘的开销。写入操作首先在缓存页中修改，该页被标记为脏页（Dirty Page），由内核的回写线程（如 flush 线程）在满足条件时统一写入磁盘。该机制处于虚拟文件系统（VFS）与块设备层之间，是存储栈的性能核心。专业工程师必须掌握，因为它直接决定读写语义、数据可靠性、内存压力和掉电一致性，也是理解数据库、消息队列、AI 训练数据管线的底层基础。

### 2. 底层原理剖析
1. 数据结构：页缓存以文件页（page）为对象，通过地址空间（address_space）组织，每个文件/设备对应一个 address_space，内含基数树（radix tree，现为 xarray）用于按文件偏移快速索引页。每个页有标志位，如 PG_dirty、PG_uptodate、PG_locked 等。
2. 读路径：读文件时，VFS 将文件偏移转换为页偏移，在页缓存中查找。命中则直接返回；未命中则分配新页，发起磁盘读，填充后插入树中并设置 PG_uptodate。此后该页可被多个进程映射共享。
3. 写路径：写文件时，VFS 找到对应页，如果不存在则分配并读取；然后将用户态数据拷贝到页中，标记 PG_dirty。write() 返回成功并不代表落盘，只是数据进入了页缓存。
4. 回写机制：内核通过以下三种方式将脏页写回：
   - 周期性回写：脏页在内存中超过 dirty_expire_centisecs 后，由 kworker/flush 线程批量写回。
   - 阈值触发：当脏页数量达到 dirty_background_ratio（内存百分比）时，后台线程开始异步写回；达到 dirty_ratio 时，写操作同步阻塞以强制回写。
   - 显式同步：用户调用 fsync()/fdatasync()/sync()，内核立即将相关脏页排队写入块设备，并等待 I/O 完成。
5. 对比前端概念：页缓存与浏览器 HTTP 缓存类似，都是通过本地副本降低访问延迟，但页缓存是内核透明的，不涉及业务语义；而 HTTP 缓存需要显式控制 Cache-Control/ETag。脏页延迟回写类似前端框架的异步批处理（如 Vue 的 nextTick），都是将频繁操作合并为一次批量提交，以降低总开销；但前端没有持久性要求，而脏页回写涉及崩溃一致性，必须有明确的刷新指令（fsync）与日志（journal）机制。

### 3. 基础代码与实战验证
```text
import os, time

def dirty_kb():
    # 读取 /proc/meminfo 中 Dirty 字段的值，单位 KB
    with open('/proc/meminfo') as f:
        for line in f:
            if line.startswith('Dirty:'):
                return int(line.split()[1])
    return -1

base = dirty_kb()
print('基准 Dirty:', base, 'KB')

fd = os.open('/tmp/test_dirty', os.O_CREAT | os.O_WRONLY, 0o644)
# 写入 4KB 数据，该数据先被拷贝到内核页缓存中的对应页，页被标记为脏
os.write(fd, b'x' * 4096)
time.sleep(0.2)
mid = dirty_kb()
print('写入后 Dirty:', mid, 'KB, 增加:', mid - base, 'KB')

# 显式调用 fsync，内核将文件页缓存中的脏页写入磁盘，并等待 I/O 完成
os.fsync(fd)
time.sleep(0.2)
after = dirty_kb()
print('fsync 后 Dirty:', after, 'KB, 减少:', mid - after, 'KB')
os.close(fd)
os.remove('/tmp/test_dirty')
```

### 4. 常见误区与进阶思考
误区1：认为 write() 返回即数据持久化。实际上 write() 仅把数据从用户缓冲区拷贝到内核页缓存，将对应页标记为 PG_dirty，I/O 延迟发生。只有 fsync()/fdatasync() 或系统自动回写才会将数据写回磁盘。这也是服务器宕机丢数据的核心原因。

误区2：认为脏页回写越快越安全。回写频率过高会导致大量小块 I/O，写放大严重，且与业务读写争抢磁盘带宽，整体吞吐下降。内核使用 dirty_background_ratio 和 dirty_ratio 双阈值，就是为了在数据安全与性能之间取得平衡。

进阶思考：如果一个文件同时被两个进程映射到各自的地址空间（MAP_SHARED），进程 A 修改页面后进程 B 立即读取，在 A 调用 msync() 之前，B 能否看到 A 的修改？为什么？进一步地，如果 A 没有调用 msync()，系统崩溃后，这个修改是否可能丢失？请从页缓存和内存映射的一致性角度分析。
