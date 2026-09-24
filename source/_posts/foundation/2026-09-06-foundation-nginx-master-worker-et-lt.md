---
title: "每日基础技术总结 · 2026-09-06 · Nginx master-worker 进程模型与 epoll ET/LT 模式差异"
date: 2026-09-06 08:00:00
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-06 · Nginx master-worker 进程模型与 epoll ET/LT 模式差异

## 📚 今日主题

> **Nginx master-worker 进程模型与 epoll ET/LT 模式差异**（后端基础）

### 1. 核心概念速览
Nginx采用master-worker多进程模型：master进程负责任务调度与worker生命周期管理，worker进程通过epoll事件驱动处理网络请求。epoll支持水平触发(LT)和边缘触发(ET)两种模式。LT模式下，只要内核缓冲区仍有可读数据，每次epoll_wait都会返回该fd；ET模式下，仅在fd状态发生变化（从无数据到有数据）时返回一次，需一次性尽量读完。本质是内核事件分发策略差异，决定了用户态事件循环的唤醒频率与处理逻辑。该机制位于操作系统I/O多路复用与Web服务器架构的交汇层，是理解高并发服务端性能底座的基石，专业工程师必须掌握以正确设计高性能服务。

### 2. 底层原理剖析
进程模型：master进程启动后解析配置，fork出固定数量的worker进程（worker_processes）。每个worker进程拥有独立的事件循环，共享监听socket（通过继承fd或SO_REUSEPORT）。master进程仅负责worker进程监控、配置reload、平滑升级，不参与请求处理。worker之间通过共享内存实现accept_mutex防止惊群，或利用SO_REUSEPORT让内核进行负载均衡。worker进程的事件循环调用epoll_wait阻塞，仅处理就绪事件。
epoll机制：epoll在内核中维护一棵红黑树管理所有被监听的fd，并维护一个就绪链表。LT（水平触发）：只要fd对应的读缓冲区非空或写缓冲区有空间，每次调用epoll_wait都会返回该fd的事件，直到缓冲区状态不再满足条件。ET（边缘触发）：仅当fd状态发生跳变（如读缓冲区从空变为非空，或写缓冲区从满变为非满）时，epoll_wait才会返回一次该fd的事件；之后即使缓冲区仍有数据，若没有新的事件到达，epoll_wait不会再返回该fd。
核心差异：LT是“状态驱动”，ET是“边沿驱动”。LT实现简单，不易漏事件，但唤醒次数多；ET要求用户态在收到事件后必须循环调用recv/read直到返回EAGAIN，否则会丢失后续事件。ET适合大并发场景减少系统调用，但编程复杂度高。
对比前端概念：这与JS事件循环中的任务队列调度类似——LT相当于每次事件循环都会检查所有挂起的任务，ET相当于只在任务状态发生跳变时触发回调。Nginx的master-worker模型类似于浏览器多进程架构：master对应browser process（管理生命周期），worker对应renderer process（处理具体事务），但Web服务器的worker通过共享监听socket接受连接，而浏览器renderer进程之间隔离更彻底。

### 3. 基础代码与实战验证
```text
以下为验证原理的极简伪代码，展示master进程fork worker + worker内部epoll ET/LT处理：
// master进程
listener_fd = socket();
bind();
listen();
for (i = 0; i < worker_processes; i++) {
    pid = fork();  // 子进程继承listener_fd
    if (pid == 0) {
        worker_loop(listener_fd);
    }
}
// master进入管理循环：waitpid、处理信号、reload等

// worker进程事件循环
worker_loop(listener_fd) {
    epfd = epoll_create1(0);
    ev.events = EPOLLIN;  // 这里可改为EPOLLIN|EPOLLET来测试ET模式
    ev.data.fd = listener_fd;
    epoll_ctl(epfd, EPOLL_CTL_ADD, listener_fd, &ev);
    while (true) {
        n = epoll_wait(epfd, events, MAX_EVENTS, -1);
        for (i = 0; i < n; i++) {
            if (events[i].data.fd == listener_fd) {
                // 接受新连接，设置为非阻塞，并注册到epoll
                conn_fd = accept(listener_fd);
                set_nonblocking(conn_fd);
                ev.events = EPOLLIN | EPOLLET; // ET模式注册
                ev.data.fd = conn_fd;
                epoll_ctl(epfd, EPOLL_CTL_ADD, conn_fd, &ev);
            } else {
                // 处理普通fd的可读事件
                while (true) {  // ET模式必须循环读直到EAGAIN
                    r = read(events[i].data.fd, buf, sizeof(buf));
                    if (r == -1 && errno == EAGAIN) {
                        break;  // 已读完，退出循环
                    }
                    if (r <= 0) break;
                    // 处理数据...
                }
            }
        }
    }
}

关键点：若将epoll事件注册为EPOLLIN（LT），则上面的while(true)内层循环非必需，因为未读完时下一次epoll_wait还会返回该fd；但若使用EPOLLET，必须通过循环读取直到EAGAIN，否则剩余数据不会再有事件触发。
```

### 4. 常见误区与进阶思考
误区1：“ET模式一定比LT高效”。实际上ET要求用户态复杂读循环，若程序错误（未读到EAGAIN）将导致永久丢失事件，且高并发下可能因为多次系统调用反而增加开销。LT在低流量下更安全且CPU开销可忽略。Nginx默认使用ET并配合完整read循环，但大多数应用更适合LT。
误区2：“master-worker模型是为了多核并行处理连接”。实际上worker进程数通常等于CPU核数，是为了避免进程频繁切换，同时事件循环在单进程内可以处理成千上万连接；多进程是为了利用多核并隔离单点故障，而非简单并发。
思考题：若使用ET模式，当内核读缓冲区中已积累100字节数据，第一次epoll_wait返回可读，用户态仅读取50字节；之后用户态继续阻塞在epoll_wait，此时再无新数据到达，请问epoll_wait会返回吗？如果此时对端再发送1字节，会返回吗？请从内核状态变化的角度分析原因。
