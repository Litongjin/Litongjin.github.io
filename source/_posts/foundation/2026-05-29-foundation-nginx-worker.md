---
title: "每日基础技术总结 · 2026-05-29 · Nginx 的 worker 进程模型与惊群处理"
date: 2026-05-29 08:00:00
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-05-29 · Nginx 的 worker 进程模型与惊群处理

## 📚 今日主题

> **Nginx 的 worker 进程模型与惊群处理**（后端基础）

### 1. 核心概念速览
Nginx 的 worker 进程模型是 master 进程负责管理（配置解析、信号处理、fork/回收 worker），worker 进程负责实际 I/O 与请求处理。每个 worker 是独立进程，拥有独立内存空间和事件循环，通过非阻塞 socket + epoll 处理海量并发。其本质是将一个进程内的多路复用扩展为多进程实例，以利用多核 CPU，同时通过共享 listen socket 实现连接分发。

惊群（thundering herd）问题：当多个 worker 进程同时对同一个 listen socket 等待可读事件时，内核会唤醒所有等待的进程/线程，但最终只有一个进程能成功 accept()，其余进程空转唤醒，导致不必要的上下文切换、CPU 抖动和缓存污染。Nginx 需要协调多个 worker 对同一监听 socket 的访问，避免惊群。

在整个计算机体系中，这属于操作系统进程调度、网络协议栈与高并发服务器设计的关键交叉点。专业工程师必须掌握它，因为它是理解几乎所有现代高并发服务器（Nginx、Redis 多线程、Envoy）底层并发模型的基础，也是调优 Nginx 性能（worker_connections、accept_mutex、reuseport）的理论前提。

### 2. 底层原理剖析
Nginx 启动流程：master 读取配置，创建监听 socket（listen fd），然后 fork 出 N 个 worker。所有 worker 继承同一个 listen fd，但各自持有独立的 epoll 实例。

每个 worker 的事件循环伪代码：

    while (true) {
        ngx_process_events_and_timers();
        // 内部调用 epoll_wait 监听已注册事件
        // 当 listen fd 可读，表示有新连接，执行 accept()
    }

惊群产生的根本原因：多个进程在 epoll_wait 上等待同一个 fd 的可读事件。在旧版 Linux（无 EPOLLEXCLUSIVE）上，内核唤醒队列中所有等待者，但 listen fd 的 accept 队列只能被一个进程消费，导致其余进程 wakeup 后确认无事件再睡眠。

Nginx 的解决方案分三层：

1. accept_mutex（默认开启）：所有 worker 在非 listen 事件上正常竞争。若要 accept 新连接，必须先抢到进程间互斥锁。抢到锁的 worker 将 listen fd 加入自己的 epoll，其他 worker 不监听该 fd。master 通过负载均衡（ngx_accept_disabled）调整每个 worker 获取锁的概率，防止某个 worker 过载。伪代码：

    if (ngx_use_accept_mutex) {
        if (ngx_trylock_accept_mutex()) {
            // 锁成功后，将 listen fd 注册到当前 epoll
            // 此时只有该 worker 能看到新连接事件
        }
    }
    // 处理事件
    // 释放锁

2. 使用 EPOLLEXCLUSIVE（Linux 4.5+）：在 epoll_ctl 注册 listen fd 时设置该标志。内核在唤醒时只唤醒等待队列中的一个进程（由内核调度策略选择，通常是最新加入的），从源头避免惊群。Nginx 在编译时检测并支持。

3. SO_REUSEPORT（Linux 3.9+）：每个 worker 独立创建自己的 listen socket，内核按四元组哈希将新连接分发到不同 socket。完全避免共享 fd 的竞争，但需要内核支持且连接分配可能不均匀。

与前端已有知识的对比：前端工程师熟知的浏览器事件循环或 Node.js 单线程模型，本质是一个进程内单个线程通过非阻塞 I/O 实现高并发；Nginx 则是多进程 + 每个进程独立事件循环。相同点是都依赖事件驱动和异步非阻塞，不同点是 Nginx 必须处理多进程间对共享资源的竞争（listen fd），而单线程模型没有进程间同步问题，但受限于单核计算能力。这类似于 Java 的接口（定义行为契约）和 TS 的接口（结构类型契约）本质都是约束，但一个在运行期有虚表分派，另一个在编译期做结构检查；Nginx 的协调机制与前端的事件循环同样都是为了在特定资源约束下最大化并发，但作用层次不同。

### 3. 基础代码与实战验证
```text
以下用极简 C 语言伪代码演示惊群问题与 accept_mutex 思想，不依赖 Nginx 内部实现。

#include <sys/socket.h>
#include <sys/epoll.h>
#include <unistd.h>
#include <stdio.h>

#define WORKER_NUM 4

int main() {
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    // bind, listen 已省略

    for (int i = 0; i < WORKER_NUM; i++) {
        pid_t pid = fork();
        if (pid == 0) {
            // 子进程：每个都独立创建 epoll 并监听同一个 listen_fd
            int epfd = epoll_create(1);
            struct epoll_event ev;
            ev.events = EPOLLIN;   // 未加 EPOLLEXCLUSIVE 时，所有 worker 都会收到唤醒
            ev.data.fd = listen_fd;
            epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &ev);

            while (1) {
                struct epoll_event events[10];
                int n = epoll_wait(epfd, events, 10, -1);
                for (int j = 0; j < n; j++) {
                    if (events[j].data.fd == listen_fd) {
                        // 多个进程同时从这里醒来，但只有一个 accept 成功
                        int conn_fd = accept(listen_fd, NULL, NULL);
                        if (conn_fd > 0) {
                            printf("Worker %d accepted connection %d\n", getpid(), conn_fd);
                        } else {
                            printf("Worker %d woke up but accept failed (EAGAIN)\n", getpid());
                        }
                    }
                }
            }
        }
    }
    while (1) pause();
}

// 运行该程序，当有客户端连接时，会观察到多个 Worker 打印 wake up，但只有一个打印 accepted。
// 这就是惊群。

// Nginx 的 accept_mutex 优化：只有获得锁的 worker 才把 listen_fd 加入 epoll，
// 伪代码：
// if (trylock(accept_mutex)) {
//     epoll_ctl(epfd, ADD, listen_fd);
//     // 处理 accept
//     epoll_ctl(epfd, DEL, listen_fd);
//     unlock(accept_mutex);
// }
```

### 4. 常见误区与进阶思考
误区1：认为 worker 进程数越多越好。实际上每个 worker 都是一个事件循环，受 CPU 核心数限制，超过核心数会导致无谓的进程调度和缓存失效。Nginx 官方建议 worker_processes = CPU 核心数。但即使这样，多进程仍然可能惊群，因为 CPU 核数多不代表内核唤醒不会广播，所以仍需要 accept_mutex 或 EPOLLEXCLUSIVE。

误区2：认为 accept_mutex 在 Linux 上永远必要。现代 Linux 内核提供了 EPOLLEXCLUSIVE 和 SO_REUSEPORT，它们可以在不引入用户态锁的情况下避免惊群。accept_mutex 反而在极端高并发下成为瓶颈（所有 worker 竞争同一把锁），所以 Nginx 在检测到支持 EPOLLEXCLUSIVE 时默认关闭 accept_mutex。

思考题：在 Linux 4.5 以下（无 EPOLLEXCLUSIVE）的系统中，如果 Nginx 同时开启 accept_mutex 和多个 worker，那么当一个 worker 持有锁并将 listen_fd 加入其 epoll 后，其他 worker 的 epoll 中并没有 listen_fd，此时新连接到达，内核会唤醒持有锁的 worker；但持有锁的 worker 在 accept 后释放锁，接着可能有另一个 worker 抢到锁。请问：在释放锁与重新添加 listen_fd 之间的窗口期，如果又有新连接到达，会发生什么？请结合 accept_mutex 的实现细节解释，为什么这个窗口期不会造成连接丢失？
