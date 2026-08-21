---
title: "每日基础技术总结 · 2026-06-20 · 零拷贝：sendfile、splice 与 DMA 引擎"
date: 2026-06-20 08:00:00
categories: [技术分享]
tags: ["技术分享", "操作系统基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-06-20 · 零拷贝：sendfile、splice 与 DMA 引擎

## 📚 今日主题

> **零拷贝：sendfile、splice 与 DMA 引擎**（操作系统基础）

### 1. 核心概念速览
零拷贝（Zero-Copy）是一类避免用户态与内核态之间多余数据复制的 I/O 优化技术，其本质是让数据在 DMA 引擎与内核缓冲区之间直接传输，或者通过页表重映射/描述符传递来减少 CPU 参与的拷贝次数。它解决的是传统 read+write 路径中数据被反复拷贝（磁盘→内核页缓存→用户缓冲→内核 socket 缓冲→网卡）导致的 CPU 开销和内存带宽浪费问题。机制核心是：利用 DMA 引擎完成设备间数据搬运，利用内核态的文件描述符传递或 splice 管道机制避免用户态参与，利用 sendfile 或 copy_file_range 将文件到 socket 的传输收敛为一次系统调用。在整个计算机体系中，零拷贝位于操作系统 I/O 栈与网络协议栈的交界处，是高性能网络服务器（Nginx、Kafka、RocketMQ）、分布式存储和数据库引擎的基石能力。专业工程师必须掌握它，因为这是理解高吞吐低延迟系统设计的前提，也是排查性能瓶颈（如 CPU 占用率高、内存带宽不足）时定位数据通路的关键；同时它涉及虚拟内存、DMA、系统调用、文件系统与网络协议栈的协同，是衡量对底层运行机制理解深度的重要标尺。

### 2. 底层原理剖析
传统文件发送路径：read(fd, buf) 引发一次系统调用，DMA 将磁盘数据拷贝到内核页缓存（Page Cache），CPU 将页缓存数据拷贝到用户态 buf；write(sockfd, buf) 再次陷入内核，CPU 将用户态 buf 拷贝到 socket 发送缓冲区，之后 DMA 将发送缓冲区数据拷贝到网卡。共涉及 4 次拷贝（2 次 DMA、2 次 CPU）和 4 次上下文切换。

sendfile 的机制：sendfile(out_fd, in_fd, offset, count) 直接在两个内核文件描述符之间传输数据。其核心是避免用户态缓冲区：DMA 将磁盘数据拷贝到页缓存；然后通过描述符（如网卡支持 SG-DMA）直接将页缓存中的数据组织成 SG 列表交给网卡，由 DMA 引擎直接拉取到网卡缓冲区，全程 CPU 不参与数据拷贝，仅负责控制。若网卡不支持 SG，则内核会进行一次 CPU 拷贝将页缓存数据合并到 socket 缓冲区，但用户态依然零参与。关键点是：sendfile 仅适用于从文件到 socket 的单向传输，且要求 out_fd 必须支持 mmap 式操作（如 socket）。

splice 的机制：splice(fd_in, off_in, fd_out, off_out, len, flags) 基于 Linux 管道（pipe）作为中转，核心是“页缓存引用转移”而非数据拷贝。它在两个文件描述符之间移动数据，其中一个必须是管道。当数据从文件 splice 到管道时，内核不复制数据，而是将页缓存中的页面指针挂入管道缓冲区；再从管道 splice 到 socket 时，同样转移页面引用。整个过程通过 VFS 层的 splice_read 和 splice_write 回调实现，如果两端都支持，则实现真正零拷贝（无 CPU 拷贝）；如果目标设备不支持（如普通文件），则退化为内核内拷贝。splice 比 sendfile 更通用，因为它不要求目标必须是 socket，可以用于任意两个文件描述符（如文件到文件、文件到终端），但一端必须是管道。

DMA 引擎的本质：DMA（Direct Memory Access）控制器是外设与内存之间传输数据的硬件引擎，它可以在不占用 CPU 的情况下完成批量数据搬运。CPU 只需初始化 DMA 描述符（源地址、目标地址、长度、方向）并启动传输，传输完成后再通过中断通知 CPU。零拷贝依赖 DMA 将设备（磁盘/网卡）与内存之间的数据搬运从 CPU 中卸载，否则即使内核态内拷贝减少，CPU 仍然会被数据搬运占满。

与前端已有概念的对比：可以把传统 I/O 路径类比为 JavaScript 中通过 JSON 序列化/反序列化在系统间传递对象——每次传递都要完整复制数据，CPU 被占用做格式转换；而零拷贝类似于通过引用传递（如传递对象的引用或使用共享内存），避免了序列化开销。更准确的类比是：前端中 React 的虚拟 DOM diff 是尽量减少实际 DOM 操作，而零拷贝是尽量减少实际内存拷贝；前者用算法优化减少昂贵的 DOM 操作，后者用内核机制减少昂贵的 CPU 拷贝。另一个对比：Java 的接口与 TypeScript 的接口区别在于，Java 接口是运行时多态契约（方法表查找），TS 接口是编译期结构类型检查；零拷贝类似地，在用户态看不到“接口”，但内核通过文件描述符抽象（VFS 接口）和管道抽象提供统一操作入口，sendfile/splice 本质是利用了内核多态（file_operations 中的 splice_read/splice_write 回调）来分派到具体设备实现，与 Java 接口的运行时多态同构，而传统 read/write 则像是总是通过一个强制序列化的通用接口传递数据。

### 3. 基础代码与实战验证
```text
以下是一个用 C 语言验证 sendfile 零拷贝行为的极简服务端程序，它接收 HTTP 请求后通过 sendfile 将文件内容直接发送到客户端 socket。代码不依赖任何框架，仅使用 POSIX 系统调用。

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <sys/sendfile.h>
#include <arpa/inet.h>

int main() {
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in addr = {
        .sin_family = AF_INET,
        .sin_addr.s_addr = htonl(INADDR_ANY),
        .sin_port = htons(8080)
    };
    bind(listen_fd, (struct sockaddr*)&addr, sizeof(addr));
    listen(listen_fd, 10);

    int file_fd = open("./bigfile.bin", O_RDONLY);
    off_t file_size = lseek(file_fd, 0, SEEK_END);
    lseek(file_fd, 0, SEEK_SET);

    while (1) {
        int client_fd = accept(listen_fd, NULL, NULL);
        // 接收并丢弃 HTTP 请求头（简化处理，仅演示 sendfile）
        char buf[1024];
        read(client_fd, buf, sizeof(buf));

        off_t offset = 0;
        ssize_t sent = sendfile(client_fd, file_fd, &offset, file_size);
        // sendfile 调用后：
        // 1. 内核从 file_fd 对应的页缓存中读取数据，若页缓存缺失，则通过 DMA 从磁盘加载到页缓存（第一次 DMA 拷贝）。
        // 2. 若网卡支持 SG-DMA，则内核构造 SG 列表直接交给网卡，由 DMA 从页缓存拉取数据发送（第二次 DMA 拷贝），无 CPU 拷贝。
        // 3. offset 被更新为实际发送字节数，file_fd 的文件偏移不变（因为传入的是指针）。
        // 整个过程只有 2 次 DMA 拷贝，无 CPU 数据拷贝，用户态仅发一次系统调用。

        close(client_fd);
    }
    return 0;
}

如果要验证 splice，可用两个 splice 调用在文件与 socket 之间通过管道中转：
int pipefd[2];
pipe(pipefd);
// 第一次 splice：将文件数据转移到管道（引用转移，无拷贝）
splice(file_fd, NULL, pipefd[1], NULL, file_size, SPLICE_F_MOVE);
// 第二次 splice：将管道数据转移到 socket（引用转移，无拷贝）
splice(pipefd[0], NULL, client_fd, NULL, file_size, SPLICE_F_MOVE);
// 注意：splice 要求两端至少一端是管道；SPLICE_F_MOVE 提示内核尝试转移页面引用而非复制，实际行为取决于文件系统。

用 strace 可以验证：strace -e sendfile,splice ./program，观察是否只发生对应的系统调用，且用户态没有 read/write 大量数据。通过对比传统 read/write 版本的 CPU 占用率（perf stat）可以量化零拷贝的优势。
```

### 4. 常见误区与进阶思考
误区 1：认为 sendfile/splice 一定零拷贝。实际零拷贝取决于硬件和内核配置：如果网卡不支持 SG-DMA（scatter-gather），sendfile 仍会有一个 CPU 拷贝将页缓存数据整理到连续 socket 缓冲区；如果文件系统不支持 splice 页面转移（如某些网络文件系统），splice 会退化为内核缓冲区拷贝。零拷贝是特定条件下达成的最优路径，不是系统调用的必然行为。

误区 2：混淆零拷贝与减少系统调用次数。零拷贝的核心是消除 CPU 参与的数据搬运（尤其用户态与内核态之间的拷贝），而系统调用次数的减少只是附带结果。有些实现（如 mmap + write）也能减少拷贝（mmap 避免一次 read 拷贝，但 write 时仍需 CPU 拷贝到 socket 缓冲区），且存在页错误和映射同步风险，因此不能简单认为 mmap 就是零拷贝。判断标准应看 CPU 是否承担了逐字节/逐块的数据复制工作。

进阶思考题：在 Linux 中，sendfile 从文件向 socket 发送数据时，如果文件在发送过程中被另一个进程截断（truncate）或写入，内核如何保证数据一致性？请从页缓存、文件锁、i_size 检查和 splice/sendfile 实现中的协作者手分析，说明在何种条件下会发生发送长度与实际数据不一致，以及为什么零拷贝路径上的数据快照机制（如 FUSE 的 writeback cache）会带来额外代价。这要求你深入理解 VFS 层和页缓存回写机制，而非停留在系统调用语义表面。
