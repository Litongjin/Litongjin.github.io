---
title: "每日基础技术总结 · 2026-09-13 · Linux epoll 的 LT（水平触发）与 ET（边缘触发）模式差异及边界条件处理"
date: 2026-09-13 08:00:00
categories: [技术分享]
tags: ["技术分享", "架构与设计"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-13 · Linux epoll 的 LT（水平触发）与 ET（边缘触发）模式差异及边界条件处理

## 📚 今日主题

> **Linux epoll 的 LT（水平触发）与 ET（边缘触发）模式差异及边界条件处理**（架构与设计）

### 1. 核心概念速览
Linux epoll 的 LT（Level Triggered，水平触发）与 ET（Edge Triggered，边缘触发）是就绪通知的两种触发语义，本质是内核在事件就绪状态变化时向用户进程投递通知的策略。LT 模式下，只要 fd 处于可读（可写）状态，每次 epoll_wait 都会返回该 fd；ET 模式下，仅当 fd 状态从未就绪变为就绪（即出现上升沿）时，epoll_wait 才返回一次该 fd。其解决的是 I/O 多路复用中事件通知频率与处理效率的权衡：LT 简单安全但重复唤醒，ET 高效精简但要求用户一次性处理完所有数据或状态，否则可能丢失后续通知。该机制位于操作系统内核 VFS 层的 waitqueue 机制之上，是高性能网络服务器（如 Redis、Nginx、Netty 的 epoll 版本）的基石。专业工程师必须掌握，因为事件驱动模型是后端高并发与 AI 推理中异步 I/O 的基础，理解 LT/ET 的边界条件直接决定代码是否能正确工作，避免死锁、饥饿或数据残留。

### 2. 底层原理剖析
epoll 在内核中为每个被监控的 fd 维护一个就绪队列。当文件描述符状态改变时，底层驱动调用 wake_up 唤醒等待队列上的 epoll 回调。LT 与 ET 的区别体现在回调中如何将 epitem 放入就绪队列：LT 模式下，只要文件状态满足事件条件（如 socket 接收缓冲区中有数据），每次 epoll_wait 扫描就绪队列后，该 fd 会被重新放回就绪队列，因此 epoll_wait 反复返回；ET 模式下，仅当状态出现“从无到有”的跳变时（如从不可读变为可读，或缓冲区中新增数据使得 poll 返回掩码从不满足变为满足），才将 epitem 放入就绪队列，且用户取出后不会自动重新加入，除非再次发生边沿跳变。

从内核伪码看：

```
// ep_poll_callback（由驱动在状态变化时调用）
if (epi->event.events & EPOLLET) {
    // ET：仅当 poll 检查到状态不满足→满足 或 有新数据到达导致掩码变化时，才列入 ready list
    if (检测到上升沿) list_add_tail(epi, ready_list);
} else {
    // LT：只要当前 poll 掩码中有兴趣的事件，就持续列入 ready list
    if (epi->poll & event_mask) list_add_tail(epi, ready_list);
}
```

用户侧的关键差异在于：LT 时 epoll_wait 会因 handle 后仍未读完数据而立即再次返回；ET 时一次返回后，即使还有未读数据，epoll_wait 也不会再返回该 fd，必须等待下一次边沿（新数据到达或可写状态变化）。因此 ET 模式强制要求使用非阻塞 I/O，并在读事件中循环 read 直至返回 EAGAIN，在写事件中循环 write 直到缓冲写满或 EAGAIN。

与前端概念对比：LT 类似于 DOM 的 mousemove 事件——只要鼠标在移动（状态持续满足），就不断触发；ET 类似于 click 事件——只有按下/释放的瞬间（边沿）触发一次。更接近的类比是 Web 上可观察对象：LT 是 BehaviorSubject，任何订阅者一进来就立即收到当前值，且值变化时持续推送；ET 是普通 Subject，只在值变化时推送快照，新订阅者不会自动收到当前值。与 Java 接口 / TypeScript 接口的本质区别在于：该对比并非类型系统契约层面的差异，而是事件分发策略的差异——LT 是状态驱动，ET 是变化驱动。理解这一点可类比前端事件委托中监听目标元素与监听 document 的区别，但更精确的是理解“就绪状态”与“就绪变化”作为唤醒源的本质不同。

### 3. 基础代码与实战验证
以下是一个极简 C 程序，演示同一个 socket 在 LT 和 ET 模式下的行为差异以及 ET 必须处理 EAGAIN 边界。

```c
#include <sys/epoll.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>
#include <stdio.h>

int main() {
    int sfd = socket(AF_INET, SOCK_STREAM, 0);
    // 设置非阻塞，ET 模式下必须使用非阻塞 I/O，因为需要循环读直到 EAGAIN
    fcntl(sfd, F_SETFL, fcntl(sfd, F_GETFL, 0) | O_NONBLOCK);

    struct sockaddr_in addr = {
        .sin_family = AF_INET,
        .sin_port = htons(8080),
        .sin_addr.s_addr = INADDR_ANY
    };
    bind(sfd, (struct sockaddr*)&addr, sizeof(addr));
    listen(sfd, 8);

    int epfd = epoll_create1(0);
    struct epoll_event ev = {0};
    ev.events = EPOLLIN;  // 注释：LT 模式；如改为 EPOLLIN | EPOLLET 则为 ET 模式
    ev.data.fd = sfd;
    epoll_ctl(epfd, EPOLL_CTL_ADD, sfd, &ev);

    while (1) {
        struct epoll_event events[1];
        int n = epoll_wait(epfd, events, 1, -1);  // 阻塞等待事件
        for (int i = 0; i < n; i++) {
            if (events[i].data.fd == sfd) {
                // 接受连接，同样设置为非阻塞，并加入 epoll
                int cfd = accept(sfd, NULL, NULL);
                fcntl(cfd, F_SETFL, fcntl(cfd, F_GETFL, 0) | O_NONBLOCK);
                struct epoll_event cev = {0};
                cev.events = EPOLLIN;  // 注意：这里演示 LT
                cev.data.fd = cfd;
                epoll_ctl(epfd, EPOLL_CTL_ADD, cfd, &cev);
            } else {
                // 读取数据
                char buf[1024];
                // LT 模式下，一次 read 后如果没有读完，epoll_wait 会再次返回；
                // ET 模式下必须循环到这里返回 EAGAIN，否则剩余数据可能永远无法被处理。
                for (;;) {
                    ssize_t r = read(events[i].data.fd, buf, sizeof(buf));
                    if (r > 0) {
                        // 处理数据...
                    } else if (r == -1 && errno == EAGAIN) {
                        break;  // 已读完，退出循环
                    } else if (r == 0) {
                        close(events[i].data.fd);
                        break;
                    }
                }
            }
        }
    }
    return 0;
}
```

如果上述代码中连接 fd 的事件设为 `EPOLLIN | EPOLLET`，则每个连接真正读取数据的时机仅出现一次边沿。若在 ET 模式下没有循环读到 EAGAIN，内核就绪状态不会产生新边沿，剩余数据会一直留在缓冲区，直到下一个数据包到达时才被一并读出——这就是：“ET 模式必须处理 EAGAIN” 的底层原因。

### 4. 常见误区与进阶思考
**常见误区 1：ET 模式比 LT 模式“效率高”是绝对的。** 实际上 ET 的收益在于减少 epoll_wait 的重复返回次数，但若用户每个事件都只处理少量数据，LT 可能因为内核从头扫描就绪队列而略有开销，但 ET 若没有做循环读，则会导致严重的数据残留。真正的效率差异取决于应用是否能够以单次边沿驱动完成所有处理。若应用逻辑要求每次事件处理工作量很小，LT 反而更简单可靠，性能足够。

**常见误区 2：在 ET 模式下使用阻塞 fd 或读到 EAGAIN 之前就退出读取循环。** ET 的语义是“通知一次”，一旦 read 返回数据但未达到 EAGAIN，说明缓冲区可能还有数据，若停止读取，这些数据将不会被新的事件通知覆盖（因为状态已经处于就绪，没有新的边沿），从而造成延迟或饥饿。正确的 ET 处理必须是“非阻塞 fd + 循环读/写直到 EAGAIN”。若误用阻塞 fd，当缓冲区暂时无数据时 read 会阻塞，整个事件循环会被卡死，丧失多路复用能力。

**深度思考题：** 假设用 EPOLLET 监听一个 socket 的可读事件，socket 接收缓冲区初始为空。此时对端发送 1 字节数据，然后不再发送。用户进程从 epoll_wait 返回后，在循环中只读取了这 1 字节，然后退出循环（未调用 read 到 EAGAIN 就退出）。然后用户进程再次调用 epoll_wait，会立即返回该 fd 吗？请从就绪队列的入队条件与边沿检测机制两个层面解释，若不会，那么这一字节数据导致的“可读状态”在系统中如何被感知？如果用户后续又在这个 fd 上调用 epoll_wait，并且缓冲区仍然有旧数据但无新数据到达，会发生什么？答案：不会立即返回。ET 的边沿检测只发生在状态从“不可读”变为“可读”的瞬间，该瞬间已触发过一次入队。即使缓冲区仍有数据，内核不会再次产生边沿，fd 不会重新入队。除非新数据到达（产生新的上升沿），或者用户通过 `epoll_ctl` 重置/重新注册该 fd 来强制触发一次状态检查。这解释了为什么 ET 模式要求用户必须在一次通知内把所有数据读完，否则只能等下一次输入。
