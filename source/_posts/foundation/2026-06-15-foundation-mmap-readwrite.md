---
title: "每日基础技术总结 · 2026-06-15 · mmap 与 read/write 的拷贝路径对比"
date: 2026-06-15 08:00:00
categories: [技术分享]
tags: ["技术分享", "操作系统基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-06-15 · mmap 与 read/write 的拷贝路径对比

## 📚 今日主题

> **mmap 与 read/write 的拷贝路径对比**（操作系统基础）

### 1. 核心概念速览
mmap 是 POSIX 提供的内存映射机制，将文件或设备映射到进程虚拟地址空间，使文件内容作为内存区域直接访问。read/write 是传统 I/O 系统调用，通过内核页缓存与用户缓冲区间的显式数据搬运完成读写。本质区别在于数据从内核态到用户态的传递方式：read/write 要求 CPU 负责一次内核缓冲区到用户缓冲区的拷贝，而 mmap 通过页表映射让进程直接访问页缓存，省去该次拷贝。该机制处于操作系统虚拟内存与文件系统的交界处，是理解零拷贝、共享内存、数据库存储引擎、AI 大模型权重加载等技术的基础。专业工程师必须掌握，因为它直接影响 I/O 路径的吞吐量、延迟与内存占用，也是性能分析和调优的核心知识。

### 2. 底层原理剖析
read 系统调用路径：进程调用 read(fd, buf, n) 陷入内核，内核检查页缓存，若数据不在则通过 DMA 从磁盘载入页缓存（第一次拷贝），随后内核将页缓存中数据复制到用户空间 buf（第二次拷贝，CPU 参与），系统调用返回，进程继续。write 类似：数据从用户空间复制到页缓存（CPU 拷贝），再通过 DMA 异步写回磁盘。整个过程至少发生两次数据拷贝和一次用户态/内核态切换。
mmap 路径：进程调用 mmap 建立文件到虚拟地址空间的映射，此时不加载数据。当进程首次访问映射区域时，触发缺页异常，内核通过 DMA 从磁盘将页面载入页缓存，并在页表中建立虚拟页到物理页（页缓存）的映射。后续访问直接由 CPU 通过地址转换访问页缓存，无需再复制数据。也就是说，mmap 将“文件数据”与“进程内存”通过页表共享，省去了 read/write 中内核到用户空间的那次 CPU 拷贝。但注意，磁盘到页缓存的 DMA 拷贝仍然存在，因此 mmap 不是零拷贝，而是减少一次拷贝。
对比前端概念：read/write 类似 JavaScript 中通过 JSON 序列化在系统间传递数据，需要将对象复制为字符串再解析，产生全量复制；mmap 类似通过 SharedArrayBuffer 或 Object 引用共享底层内存，多个执行者直接访问同一份数据，无需序列化。从工程角度，read/write 适合数据量小或需要修改隔离的场景；mmap 适合大文件随机访问或跨进程共享，但需注意并发一致性和映射生命周期。

### 3. 基础代码与实战验证
```text
下面以读取文件全部内容为例，展示 mmap 与 read 的差异（省略错误处理）：
#include <fcntl.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <unistd.h>

void mmap_read(const char *path) {
    int fd = open(path, O_RDONLY);
    struct stat st;
    fstat(fd, &st);
    size_t len = st.st_size;
    // mmap 建立映射，不拷贝数据；返回的地址直接映射到页缓存
    char *map = mmap(NULL, len, PROT_READ, MAP_PRIVATE, fd, 0);
    // 第一次访问 map[0] 触发缺页中断，DMA 从磁盘读入页缓存并建立页表映射
    volatile char c = map[0];
    // 后续访问 map[i] 均直接访问页缓存，无用户态与内核态之间的数据拷贝
    munmap(map, len);
    close(fd);
}

void read_all(const char *path) {
    int fd = open(path, O_RDONLY);
    struct stat st;
    fstat(fd, &st);
    size_t len = st.st_size;
    char *buf = malloc(len);
    // read 系统调用将页缓存中的内容复制到 buf，这是一次 CPU 拷贝
    ssize_t n = read(fd, buf, len);
    // 如果页缓存未命中，内核还会先从磁盘 DMA 读入页缓存
    free(buf);
    close(fd);
}
注释已说明底层运作。注意：mmap 返回后，数据仍在磁盘；访问时才触发缺页加载，这是其按需分页的特性。
```

### 4. 常见误区与进阶思考
误区1：认为 mmap 总是比 read/write 快。实际上，mmap 每次访问映射页可能触发缺页异常，对于小文件或顺序读，系统调用和缺页处理开销可能超过节省的拷贝成本；且 mmap 会占用地址空间，多线程并发访问时可能引发 TLB 抖动。应结合访问模式和数据规模评估。
误区2：认为 mmap 是零拷贝。mmap 仍有一次 DMA 从磁盘到页缓存的拷贝，只是消除了内核到用户空间的 CPU 拷贝。真正的零拷贝需借助 sendfile、splice 或进程间共享内存等手段。
思考题：在多进程同时以 MAP_SHARED 方式映射同一文件时，若一个进程写入数据，另一个进程立即读取，该数据是否会实时可见？请结合页缓存、页表映射和内存屏障机制分析可见性条件。
