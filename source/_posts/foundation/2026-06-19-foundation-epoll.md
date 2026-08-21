---
title: "每日基础技术总结 · 2026-06-19 · epoll 的水平触发与边缘触发模式"
date: 2026-06-19 08:00:00
categories: [技术分享]
tags: ["技术分享", "操作系统基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-06-19 · epoll 的水平触发与边缘触发模式

## 📚 今日主题

> **epoll 的水平触发与边缘触发模式**（操作系统基础）

### 1. 核心概念速览
epoll的水平触发（LT）与边缘触发（ET）是内核向用户空间交付就绪事件的两种模式。LT：只要文件描述符（fd）处于就绪状态（如接收缓冲区有数据），每次epoll_wait都会将其从就绪列表中返回；ET：仅当fd的就绪状态发生从无到有的边沿跃迁时（例如新数据到达，或缓冲区从满变为有空间），才将该fd加入就绪列表并返回一次。本质是状态通知与事件通知的区别，对应数字电路中电平触发与边沿触发的概念。它解决的问题：在I/O多路复用场景中，控制事件通知的粒度与次数，以平衡系统调用开销与应用处理效率。该知识点位于操作系统I/O子系统，是理解Linux高性能网络服务（如Nginx、Redis）的基础。专业工程师必须掌握，因为其直接影响事件循环的设计、缓冲区处理策略和并发模型正确性，也是排查性能问题的关键。

### 2. 底层原理剖析
内核实现：每个epoll实例维护一个就绪链表（rdllist）和一棵红黑树（存储关注fd）。当fd就绪（如数据到达），驱动调用ep_poll_callback，将fd加入就绪链表。LT模式在epoll_wait返回前，会检查该fd是否仍处于就绪状态，若是则保留在就绪链表中，因此下一次epoll_wait仍会返回。ET模式则会在返回时将该fd从就绪链表中移除，除非后续再次发生状态跃迁（如又有新数据到达），否则即使缓冲区仍有数据，也不再通知。本质上，LT是在每次wait时重新评估状态；ET是在状态跃迁时产生一次性事件。
与前端概念对比：LT类似于RxJS的BehaviorSubject——订阅时立即获得当前值，只要值存在就推送；ET类似于Subject——只推送订阅之后产生的新值，不保留旧状态。另一个对比：DOM的change事件（仅在值最终改变且失焦时触发）类似ET，而input事件（每次输入都触发）类似LT，但更本质的是事件是否携带并保留当前状态。
流程（伪代码）：
1. 应用调用epoll_ctl注册fd并指定LT或EPOLLET。
2. 当fd状态变化（例如TCP接收数据），内核唤醒等待队列，执行回调。
3. LT：回调将fd放入rdllist；epoll_wait遍历rdllist并返回所有fd；如果fd仍就绪（如数据未读完），则保持在rdllist中（或重新插入）。
4. ET：回调仅当fd尚未在rdllist中时放入；epoll_wait返回后，立即将fd从rdllist中移出。若应用未读完数据，fd不再在rdllist中，因此不会再次返回；只有下次新的状态跃迁（新数据到达）才再次加入。
注意：对于EPOLLET，如果使用水平触发逻辑处理事件（如每次只read一次），会导致数据滞留，因为后续没有通知。因此ET模式通常要求应用循环调用read/write直到EAGAIN（非阻塞I/O）。

### 3. 基础代码与实战验证
```text
以下为纯C伪代码，展示LT与ET在未处理完数据时的差异（使用管道）：

int epfd = epoll_create(1);
struct epoll_event ev; int pipefd[2]; pipe(pipefd);

// LT模式（默认）
ev.events = EPOLLIN; ev.data.fd = pipefd[0];
epoll_ctl(epfd, EPOLL_CTL_ADD, pipefd[0], &ev);
write(pipefd[1], buffer, 1);  // 写入一个字节
epoll_wait(epfd, &ev, 1, -1); // 第一次返回，可读
read(pipefd[0], buffer, 1);   // 读走一个字节，无剩余
write(pipefd[1], buffer, 2);  // 再写入两个字节
epoll_wait(epfd, &ev, 1, -1); // 返回，read一个字节
epoll_wait(epfd, &ev, 1, -1); // 再次返回，因为仍有一个字节未读

// ET模式
struct epoll_event ev2; ev2.events = EPOLLIN | EPOLLET; ev2.data.fd = pipefd[0];
epoll_ctl(epfd, EPOLL_CTL_ADD, pipefd[0], &ev2);
write(pipefd[1], buffer, 2);  // 写入两个字节
epoll_wait(epfd, &ev2, 1, -1); // 返回一次
read(pipefd[0], buffer, 1);   // 只读一个字节，仍有一个字节剩余
epoll_wait(epfd, &ev2, 1, -1); // 不会返回，因为无新数据到达，且该fd已从就绪链表移除

注释：LT每次wait检查fd就绪队列，只要有未读数据就返回；ET在wait返回后将该fd从就绪队列移除，除非有新的状态变化（新数据）才会再次加入。
```

### 4. 常见误区与进阶思考
常见误区：
1. 认为ET模式下，如果一次没有读完数据，剩余数据会被内核丢弃。实际数据仍留在内核缓冲区，只是不再产生通知，直到新数据到达产生新的边沿。
2. 认为ET一定优于LT。实际LT更安全、更简单，ET需要严格配合非阻塞I/O和循环读写，否则容易漏事件；ET减少的是事件通知次数，但要求应用在一次事件中处理完所有可用数据，否则剩余数据不会再次触发。
思考题：在ET模式下，如果一次epoll_wait返回后，应用只read了一部分数据，然后不再次read，而是等待下一次epoll_wait。请问会发生什么？为什么？请从内核就绪链表的状态变化分析。
