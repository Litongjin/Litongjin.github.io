---
title: "每日基础技术总结 · 2026-09-10 · Go 的 GMP 调度器与 work stealing"
date: 2026-09-10 08:00:00
categories: [技术分享]
tags: ["技术分享", "编程语言底层"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-10 · Go 的 GMP 调度器与 work stealing

## 📚 今日主题

> **Go 的 GMP 调度器与 work stealing**（编程语言底层）

### 1. 核心概念速览
Go 的 GMP 调度器是 Go runtime 对用户态 goroutine（G）在 OS 线程（M）上多路复用的核心机制。P（Processor）是虚拟出的调度上下文，持有本地可运行队列 runq 和 runnext 指针；只有绑定 P 的 M 才有资格执行 G。其本质是 N:M 调度：N 个 G 被运行时调度器映射到 M 个 OS 线程上，P 的数量由 GOMAXPROCS 限定，决定了任意时刻最多有多少个 G 并行执行。它要解决的是：让数量极大的轻量级协程在小规模的 OS 线程上并发运行，兼顾抢占、阻塞、多核利用与负载均衡。work stealing 是一种去中心化的负载均衡策略：当某个 P 的本地队列变空时，它从其他 P 的本地队列尾部批量拿取 G，以最小化闲置资源并减少全局锁竞争。GMP 位于语言运行时与操作系统内核之间，是所有 Go 并发程序的底层地基；理解它是排查性能瓶颈、解释调度延迟、设计高吞吐后端服务以及阅读 AI 训练框架 Go 侧代码的前提。

### 2. 底层原理剖析
底层运行机制：
1. G 的生命周期与队列：运行时维护一个全局可运行队列（带锁的 FIFO）和每个 P 的本地队列（无锁环形队列，容量 256）。新建 goroutine 一般先放在 runnext，若 runnext 非空则压入本地 runq 尾部；本地 runq 满时会把一半 G 搬移到全局队列。
2. M 的调度循环：每个 M 循环执行 schedule()，优先从 P.runnext 取 G，其次从本地 runq 头部 pop；本地为空时，按约 1/61 的节奏从全局队列 pop，防止全局队列饿死；全局也为空则进行 work stealing——随机挑选 victim P，从 victim.runq 的尾部偷走一半 G。为什么从尾部？因为 owner 从头部取 G，thief 从尾部取能降低两个 M 对同一 runq 的原子写竞争，并保持 owner 的热点 G 不被抢走。若 steal 不到任何工作，则检查定时器与 netpoll；仍无则 M 进入休眠，由 sysmon 或新事件唤醒。
3. 抢占：Go 1.14 起 scheduler 通过 OS 信号 SIGURG 实现 async preemption，让长时间运行的 G 也被打断放回队列，解决之前只能依赖函数栈增长检查的协作式抢占缺陷。
4. syscall 阻塞：当 G 进入阻塞 syscall 时，M 会被 OS 阻塞；runtime 将 P 从该 M 上摘除并交给/创建一个新 M，使 P 不浪费。syscall 返回后，G 被重新放回可运行队列，原 M 尝试重新获取 P，否则休眠。
与前端概念的对比：JS 的事件循环本质是单线程、非抢占、合作式队列（macrotask/microtask），Web Worker 是独立线程但没有共享 G 与 M 级的统一运行时调度；Go 的 GMP 是抢占式、多线程、有工作窃取的混合调度。理解差异的方法与区分 Java 的 interface 和 TS 的 interface 一致：前者是运行期真实存在的类型多态对象，后者只是编译期结构的类型约束；JS 的并发只存在于单线程队列语义，而 Go 的并发包含可并行的运行时调度实体，不能因为都叫队列或都叫并发而等价。

### 3. 基础代码与实战验证
```text
下面的 Go 代码演示了 P 与 runtime.Gosched 的关系。将 GOMAXPROCS 从 1 改为 2 可观察并行度差异；配合 GODEBUG=schedtrace=1000 可看到 runqueue 和每个 P 的本地队列长度，从而观测 work stealing 效果。

    package main

    import (
        "fmt"
        "runtime"
        "sync"
    )

    func main() {
        runtime.GOMAXPROCS(2) // 创建 2 个 P；超过 2 个 G 必须排队或 steal

        var wg sync.WaitGroup
        for i := 0; i < 4; i++ {
            wg.Add(1)
            go func(id int) {
                defer wg.Done()
                for j := 0; j < 2; j++ {
                    runtime.Gosched() // 当前 G 放入全局 runq，并触发 schedule() 重新选择 G
                    fmt.Println(id, j)
                }
            }(i)
        }
        wg.Wait()
    }

调度器核心伪代码（精确化 schedule/steal 流程）:

    func schedule() (g G) {
        if g = p.runnext; g != nil { // runnext 中的 G 优先执行（刚由 go 创建）
            p.runnext = nil
            return g
        }
        for {
            if g = p.runq.popHead(); g != nil { // 本地队列 FIFO 头部
                return g
            }
            if ticks%61 == 0 && globalRunq.len() > 0 { // 平滑全局队列
                return globalRunq.pop()
            }
            g = stealWork() // 随机选其他 P，从 victim.runq 尾部偷了一半
            if g != nil { return g }
            g = netpoll()   // 网络/定时器就绪的 G
            if g != nil { return g }
            m.stopAndPark() // 无工作，M 睡眠并等待唤醒
        }
    }
```

### 4. 常见误区与进阶思考
常见误区：
1. 认为 GOMAXPROCS 限制的是 OS 线程 M 的数量。实际 GOMAXPROCS 限制的是 P 的数量，即允许同时执行 G 的并行度；M 数量可因阻塞 syscall、sysmon、cgo 等动态增加并超过 GOMAXPROCS。误判此点会导致对 CPU 并发度、线程数的理解和调试完全偏航。
2. 认为 goroutine 调度是纯用户态协作、不经过操作系统。Go 1.14 的 async preemption 依赖 OS 信号 SIGURG 打断运行中的 M，说明当 G 占用 M 过久时，runtime 需要内核信号参与抢占；阻塞 syscall 也会让 M 被内核阻塞，需要 P 在线程之间移交。GMP 不是脱离 OS 的纯协程调度，而是用户态调度器与内核调度器的深度协作。

进阶思考题：设 GOMAXPROCS=2，P1 本地队列为空，P2 本地队列非空。work stealing 会从 P2 队列的哪一端偷取 G？为什么选择这一端对 FIFO 公平性、缓存局部性和队列竞争分别带来什么影响？如果所有 P 的本地队列和全局队列都为空，为什么还要检查 netpoll 和 timer？
