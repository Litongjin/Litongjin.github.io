---
title: "每日基础技术总结 · 2026-09-16 · Go 的 GMP 调度器与 work stealing"
date: 2026-09-16 08:00:00
categories: [技术分享]
tags: ["技术分享", "编程语言底层"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-16 · Go 的 GMP 调度器与 work stealing

## 📚 今日主题

> **Go 的 GMP 调度器与 work stealing**（编程语言底层）

### 1. 核心概念速览
GMP 是 Go runtime 的用户态 M:N 调度模型：G 是 goroutine，包含栈、状态、调度上下文；M 是内核线程（machine），是执行 Go 代码和系统调用的实体；P 是逻辑处理器（processor），是执行 Go 代码所需的资源与调度上下文，持有本地可运行队列 runq、runnext、mcache、timer 等。任一 M 只有绑定 P 才能执行 G。GOMAXPROCS 决定 P 的数量，即同时执行 Go 代码的并行度上限。Work stealing 是当某 P 的本地队列为空时，从全局队列或其他 P 的本地队列窃取可运行 G 的负载均衡机制。它解决的是：在少量 OS 线程上复用大量 goroutine、在多核上并行、在阻塞系统调用/网络 I/O/GC 时保持 P 不闲置、以及动态平衡各 P 的负载。位置：位于语言 runtime 与 OS 线程/CPU 之间，是 Go 并发性能、延迟、伸缩性的核心；AI infra 中大量并发请求、数据加载、推理任务编排都依赖它。

### 2. 底层原理剖析
1. 实体：
G：runtime.g，状态 _Grunnable/_Grunning/_Gsyscall/_Gwaiting/_Gdead 等；初始栈 2KB，按需增长；可迁移。
M：内核线程，执行态必须绑定 P；有 g0 调度栈、curg 当前 G、spinning 自旋标志。
P：逻辑处理器，数量=GOMAXPROCS；持有 runq[256] 环形队列、runnext 优先槽、mcache、timers、GC 相关状态。
2. 调度循环：schedule() -> findRunnable()。
先取 runnext；再取本地 runq 头部；每 61 个 tick 检查一次全局 sched.runq，避免全局锁竞争；再非阻塞 netpoll；然后进入 work stealing；最后 stopm/park。
3. Work stealing：
findRunnable 中最多进行 4 轮，每轮随机选 victim P。若 victim.runqhead != runqtail 或允许偷 runnext，则 runqgrab 从 victim 本地队列头部抓取约一半 G（n = n - n/2），通过 CAS 推进 victim.runqhead；偷到的 G 放入当前 P 的本地队列，并返回一个立即执行。前几轮不偷 runnext，避免破坏新建 G 的局部性；后续轮次允许偷 runnext。若本地队列满，runqputslow 会把一半 G 加当前 G 批量放入全局队列。
4. 阻塞与抢占：
系统调用：entersyscall 将 P 状态置 _Psyscall 并解绑 M；sysmon 发现 syscall 超过阈值会 retake P，交给其他 M；exitsyscall 尝试重新绑定 P，失败则把 G 放入全局队列并 park M。
网络 I/O：netpoller 将未就绪的 G 置 _Gwaiting，M 释放 P；就绪后 netpoll 返回 G 列表。
抢占：sysmon 检测到 G 运行超过 10ms 发送 SIGURG，触发异步抢占；G 在安全点保存现场，状态置 _Gpreempted，重新入队。
5. 与前端对比：
JS 主线程是单线程事件循环，只有一个调用栈和 macrotask/microtask 队列，async/await 是协作式让出，无法在多个核上并行执行 JS 代码；Web Worker 是独立线程，靠消息传递。Go 的 GMP 是 M:N 抢占式调度，G 可并行运行在多个 P/M 上，可被抢占和迁移；本地 runq + 全局 runq + work stealing 对应的是多核负载均衡，而事件循环对应的是单线程任务队列。因此前端工程师不能把 goroutine 理解为 Promise：Promise 是单线程内的异步编排，goroutine 是 runtime 管理的可并行执行单元。与 Java 21 虚拟线程类似都是 M:N，但 Go 的 P 是独立调度资源，work stealing 和 netpoller 集成在 runtime。

### 3. 基础代码与实战验证
```text
package main

import (
    "fmt"
    "runtime"
    "sync"
    "time"
)

func main() {
    runtime.GOMAXPROCS(4) // 设置 P 的数量，不是 OS 线程数
    fmt.Println("NumCPU:", runtime.NumCPU(), "GOMAXPROCS:", runtime.GOMAXPROCS(0))

    var wg sync.WaitGroup
    wg.Add(1) // 先保证计数非零，避免 Wait 与 Add 的边界竞争
    go func() {
        defer wg.Done()
        for i := 0; i < 10000; i++ {
            wg.Add(1)
            go func(id int) {
                defer wg.Done()
                time.Sleep(20 * time.Millisecond) // G 进入 _Gwaiting，M 释放 P，其他 P 可继续调度
                start := time.Now()
                for time.Since(start) < 2*time.Millisecond {
                } // 纯 CPU 段，观察 G 在不同 P 上被调度
                _ = id
            }(i)
        }
    }()
    wg.Wait()
}

运行：GODEBUG=schedtrace=1000 go run main.go
输出中 SCHED 行的 runqueue=[...] 是每个 P 的本地队列长度；若某 P 队列过长后迅速被其他 P 拉平，说明 work stealing 生效。也可用 go tool trace 观察 G 在 P 间的迁移。
```

### 4. 常见误区与进阶思考
误区一：GOMAXPROCS 等于 OS 线程数或 CPU 核数。实际上它只限制同时执行 Go 代码的 P 数量；M 可以因 syscall、cgo、阻塞系统调用远多于 P，P 也不等于物理核。
误区二：goroutine 阻塞一定会阻塞 P。普通 netpoller 网络 I/O 和大部分系统调用会释放 P；但 cgo 调用、某些非托管阻塞、大量锁竞争可能占住 M/P，导致线程膨胀或延迟。另一个常见错误是依赖 goroutine ID 或线程本地存储做上下文传递；G 会迁移，栈会增长，必须显式传参或用 context。
思考题：work stealing 从 victim 的本地 runq 头部偷取一半 G，而不是从尾部偷取。结合 runq 的生产者从尾部入队、消费者从头部出队，以及本地性和公平性，解释为什么从头部偷取更合理？如果改成从尾部偷取，会对新建 G 的 runnext 局部性、victim 自身的缓存命中、以及长任务公平性产生什么影响？
