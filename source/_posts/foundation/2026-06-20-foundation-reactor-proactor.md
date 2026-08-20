---
title: "每日基础技术总结 · 2026-06-20 · Reactor 与 Proactor 模式：事件分发差异"
date: 2026-06-20 08:00:00
categories: [技术分享]
tags: ["技术分享", "操作系统基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-06-20 · Reactor 与 Proactor 模式：事件分发差异

## 📚 今日主题

> **Reactor 与 Proactor 模式：事件分发差异**（操作系统基础）

### 1. 核心概念速览
Reactor 与 Proactor 是两种事件分发模式，用于处理并发 I/O 事件。Reactor 基于同步非阻塞 I/O：事件循环通过多路复用器（select/poll/epoll）监听文件描述符的就绪状态，当就绪事件发生时，由应用代码执行实际的 read/write 操作。Proactor 基于异步 I/O：应用发起异步读写请求（如 IOCP、io_uring），内核或系统代理完成整个 I/O 操作，随后将完成事件投递到完成队列，事件循环分发完成回调。本质区别：I/O 操作由谁执行——Reactor 由应用执行，Proactor 由系统执行。该模式位于操作系统 I/O 模型与事件驱动架构的交叉点，是 Nginx、Redis、Node.js 等高并发服务器底层架构的基础。专业工程师必须掌握，因为它是理解异步非阻塞编程、I/O 多路复用、高性能网络服务设计的基石。

### 2. 底层原理剖析
Reactor 的核心是“就绪通知”机制。伪代码如下：
reactor_loop() {
  while (true) {
    events = select(registered_fds);   // 阻塞等待就绪事件
    for (e in events) {
      handler = registry[e.fd];        // 根据 fd 找到回调
      handler(e);                      // 回调中执行非阻塞 read/write
    }
  }
}
Proactor 的核心是“完成通知”机制。伪代码如下：
proactor_loop() {
  async_read(fd, buffer, callback);    // 提交异步读，内核负责将数据读入 buffer
  while (true) {
    completion = completion_queue.wait(); // 阻塞等待完成事件
    callback = registry[completion.op];   // 根据操作找到回调
    callback(completion.result);          // 回调直接处理已就绪的数据
  }
}
关键差异在于事件触发点：Reactor 在 I/O 可操作时触发（如可读、可写），Proactor 在 I/O 操作完成时触发（如读完成、写完成）。因此 Proactor 需要内核提供真正的异步 I/O 原语，而 Reactor 只需多路复用器。对比前端概念：前端中的“事件监听”模式与 Reactor 类似——addEventListener 注册回调，事件发生时事件循环调用回调，回调中需自行获取事件数据；而“Promise/async-await”模式与 Proactor 类似——fetch 返回 Promise，异步操作完成时自动 resolve，回调直接拿到结果。异同点：相同之处在于两者都通过注册回调实现非阻塞 I/O；不同之处在于回调触发时机及数据获取方式。Reactor 回调需要主动执行 I/O，Proactor 回调直接获得完成结果。这也解释了为何前端中 Promise 通常被称为“异步结果”，而事件监听被称为“事件流”。

### 3. 基础代码与实战验证
```text
由于真实验证 Proactor 需要特定内核 API，以下用伪代码展示两种模式的本质差异。
Reactor（基于 select）：
# Reactor: 应用收到可读事件后自行 read
import select, os

def reactor_loop(fds):
    while True:
        readable, _, _ = select.select(fds, [], [])  # 阻塞等待就绪 fd
        for fd in readable:
            # 此刻 fd 一定可读，read 不会阻塞（同步非阻塞）
            data = os.read(fd, 1024)
            handle_data(fd, data)
Proactor（基于 io_uring 概念）：
# Proactor: 内核完成读操作后投递完成事件
def proactor_loop(ring, fd, buffer):
    # 提交异步读请求，内核负责将数据从 fd 读入 buffer
    io_uring_prep_read(ring, fd, buffer, len(buffer))
    io_uring_submit(ring)
    while True:
        # 阻塞等待完成队列中的事件，此时 buffer 已被内核填充
        cqe = io_uring_wait_cqe(ring)
        # 应用直接处理 buffer 中的数据，无需再调用 read
        handle_data(fd, buffer, cqe.res)
        io_uring_cqe_seen(ring, cqe)
从代码可见，Reactor 的事件循环中仍有显式的 read 调用，而 Proactor 中 read 由内核在 submit 时执行，事件循环只等待完成通知。这正是两种模式的分界线。
```

### 4. 常见误区与进阶思考
误区一：将 Reactor 视为异步 I/O。Reactor 使用的是同步非阻塞 I/O，因为读写操作由应用在事件循环中执行，只是由于事件已就绪而不会阻塞。真正的异步 I/O 指操作系统执行 I/O 并通知完成，即 Proactor。
误区二：认为 Proactor 一定优于 Reactor。Proactor 减少了应用参与 I/O 的 CPU 开销，但要求内核提供异步原语（如 IOCP、io_uring），并且可能增加内存拷贝开销。在 Linux 上，传统上 epoll 的 Reactor 模式更成熟，而 io_uring 出现后才提供高效的 Proactor 实现。选择哪种模式取决于平台和业务场景。
思考题：在 Proactor 模式中，当应用发起一个异步 write 操作时，内核何时触发完成回调？是数据已经写入 socket 发送缓冲区，还是已经通过 TCP 协议发送到对端并收到 ACK？这对应用层设计（如关闭连接、内存释放）有什么影响？
