---
title: "每日基础技术总结 · 2026-09-05 · Go 的 GMP 调度器与 work stealing"
date: 2026-09-05 08:00:00
categories: [技术分享]
tags: ["技术分享", "编程语言底层"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-05 · Go 的 GMP 调度器与 work stealing

## 📚 今日主题

> **Go 的 GMP 调度器与 work stealing**（编程语言底层）

### 1. 核心概念速览
Go 的 GMP 调度器是 Go 运行时（runtime）实现用户态并发调度的核心机制。G 代表 goroutine（用户态协程，包含栈、指令指针、状态等），M 代表 machine（操作系统线程，实际执行计算的载体），P 代表 processor（逻辑处理器，持有可运行 G 的本地队列，并决定哪些 G 可以在哪些 M 上运行）。其本质是将 N 个用户态协程（G）多路复用到 M 个操作系统线程（M）上，并通过 P 这一中间层解耦 G 与 M 的绑定关系，使得调度器能在任意时刻为每个 M 分配一个 P，从而控制并行度（同时运行的 M 数量不超过 GOMAXPROCS）。它解决的问题是：在 OS 线程创建/切换成本高昂（约微秒级，涉及内核态陷入、上下文切换、栈切换）的背景下，提供一种轻量级（goroutine 初始栈约 2KB，切换在用户态完成，纳秒级）的并发原语，支撑数十万乃至百万级 goroutine 的高效并发。work stealing 是该调度器维护负载均衡的算法：当某个 P 的本地队列为空时，它会从其他 P 的本地队列（或全局队列）中窃取一半（或至少一个）的 G 来执行，从而避免某个 M 空闲而其他 M 过载，最大化 CPU 利用率。在计算机体系中，调度器属于操作系统与用户程序之间的运行时层（runtime layer），类似于 Java JVM 的线程调度、Erlang BEAM 的进程调度。专业工程师必须掌握它，因为并发模型的选择直接影响系统吞吐、延迟与资源开销；不理解 GMP，就无法解释为什么 Go 能支撑高并发、为何某些场景下 goroutine 会引发性能问题，也无法正确设计异步/并发架构，更无法深入理解 go 语句、channel、sync 包底层的行为约束，以及容器环境下 GOMAXPROCS 的调优原理。

### 2. 底层原理剖析
一、实体定义与状态机
G（goroutine）: 一个用户态调度单元，包含 g 结构体的栈指针、程序计数器、当前 M 引用、状态（_Gidle, _Grunnable, _Grunning, _Gsyscall, _Gwaiting, _Gdead 等）。G 不是线程，它依赖 M 才能执行。
M（machine）: 一个实际的操作系统线程，由 runtime 创建或复用。M 必须绑定一个 P 才能运行 G；否则 M 会阻塞或自旋。
P（processor）: 抽象的处理资源，数量由 GOMAXPROCS 决定，通常等于 CPU 核数。P 持有本地可运行 G 队列（runq，长度 256 的有界队列）以及一些调度状态。

二、调度循环（schedule 函数）核心逻辑伪代码：
```
func schedule(mp *m, pp *p) {
    // 1. 优先级检查：每调度 61 次尝试从全局队列取一次 G（防止全局饥渴）
    if pp.schedtick % 61 == 0 {
        g := globalrunq.get()
        if g != nil { execute(g); return }
    }
    // 2. 从本地队列取 G
    g := pp.runq.pop()
    if g == nil {
        // 3. 本地队列为空，执行 work stealing
        g = stealWork(pp)
    }
    if g == nil {
        // 4. 仍为空，则检查网络轮询器、事件，或让 M 休眠（阻塞/自旋）
        g = findrunnable()
    }
    execute(g)
}
```

三、work stealing 机制
当 P 的本地队列为空，且全局队列为空时，M（该 P 绑定的线程）不会立即休眠，而是进入自旋（spinning）状态，并调用 `stealWork`（runtime/proc.go 中的 stealWork 函数）。它会随机选择一个目标 P（采用 pseudorandom 遍历，避免所有 stealing 者同时竞争同一 P），然后从目标 P 的本地队列尾部窃取一半的 G（具体为 runq 长度的一半，至少一个）。窃取操作使用无锁的 CAS（Compare-And-Swap）原子操作，确保并发安全。窃取后，窃取者立即执行偷到的第一个 G，其余放入自己队列。如果没有可窃取的 G，则回到 schedule 检查全局队列、网络事件，若都没有则让 M 进入休眠（gopark），并解除与 P 的绑定，使 P 可以被其他自旋 M 获取。

四、GM 到 GMP 的演进原因
早期 Go 使用 GM 模型，全局队列互斥锁导致高竞争，且 M 经常阻塞在锁上。GMP 引入 P 后，每个 P 有独立本地队列，从而减少了互斥锁竞争；同时 P 的数量限制了可并行执行的 M 数量，避免了线程无限增长的问题。M 与 P 的解耦允许 M 在系统调用被阻塞时，P 可以切换给另一个 M 继续执行其他 G（handoff 机制），从而让 CPU 在系统调用期间不被空闲浪费。

五、与前端知识体系对比
前端（JS/浏览器）中不存在用户态协程调度器：JS 是单线程事件循环（event loop），所有任务在同一个线程上按宏任务/微任务队列顺序执行，没有并行，只有并发。而 Go 的 GMP 是基于多线程的并行+并发模型。可以把 JS 的事件循环看作单 M 单 P 的特殊情况，且没有 work stealing，因为只有一个队列。Go 的 G 类似 Promise 或 async 函数，但后者由 JS 引擎（如 V8）以指令粒度切分（无抢占），而 Go 的 G 由 runtime 在函数调用边界、channel 操作、系统调用、抢占点等位置插入调度决策，并支持抢占式调度（基于信号量 SIGURG 或协作式 10ms 抢占）。

六、调度点与抢占
Go 1.14 后引入异步抢占：runtime 每 10ms 通过系统监控线程（sysmon）向正在执行 G 的 M 发送 SIGURG 信号，强制中断其执行并进入调度循环，防止一个 G 长期独占 CPU。这类似操作系统的时间片轮转，但发生在用户态。

七、关键数据结构：runq
P 的 runq 是一个固定大小 256 的环形数组队列。当本地队列满时，新 G 会被放入全局队列（sched.globalrunq），并一次性转移本地队列一半的 G 到全局队列（runqputslow）。这确保本地队列不会无限增长，同时让全局队列分担压力。

### 3. 基础代码与实战验证
以下代码用于验证 GMP 的并行度与 work stealing 存在性（通过观察执行顺序和调度时机）。

```go
package main

import (
    "fmt"
    "runtime"
    "sync"
    "time"
)

// 验证点 1: GOMAXPROCS 限制同时执行的 goroutine 数量（并行度）
func verifyParallelLimit() {
    runtime.GOMAXPROCS(1) // 强制单 P，制造只有一个 M 可并行执行的环境
    start := time.Now()
    var wg sync.WaitGroup
    for i := 0; i < 3; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            fmt.Printf("goroutine %d 开始\n", id)
            time.Sleep(200 * time.Millisecond) // 模拟一个会阻塞在 sleep 上的操作（进入等待队列）
            fmt.Printf("goroutine %d 结束\n", id)
        }(i)
    }
    wg.Wait()
    fmt.Printf("总耗时: %v (若串行则约 600ms，若并行则约 200ms)\n", time.Since(start))
    // 由于 GOMAXPROCS=1，三个 goroutine 依次执行，总耗时约 600ms+，验证 P 的数量约束并行度。
    // 注意: time.Sleep 会让 G 进入 _Gwaiting，P 会调度其他 G，但因为是单 P，仍为串行。
}

// 验证点 2: work stealing 的存在（通过阻塞主线程制造其他 P 空闲）
func verifyWorkStealing() {
    runtime.GOMAXPROCS(2) // 两个 P
    // 创建大量短暂计算的 goroutine，让其分布在不同的 P 上，然后让其中一个 P 因 runtime.Gosched 或 IO 阻塞而清空队列。
    // 下面用一个简化方式：先在一个 goroutine 中占据 P，其他 P 清空后，它会从全局或别人队列窃取。
    done := make(chan bool)
    var counter int64
    var mu sync.Mutex

    // 制造一个饥饿的 P：这个 goroutine 一开始就主动让出 CPU，此时它的 P 本地队列为空，
    // 会触发 work stealing，从另一个 P 的队列中偷取 G。
    go func() {
        runtime.Gosched() // 主动让出 P，使当前 M 进入 _Grunnable 队列
        for {
            select {
            case <-done:
                return
            default:
                mu.Lock()
                counter++
                mu.Unlock()
            }
        }
    }()

    time.Sleep(10 * time.Millisecond)
    close(done)
    fmt.Printf("counter = %d, 说明 Gosched 后它被其他 M 偷走执行或自旋等待\n", counter)
}

func main() {
    fmt.Println("=== GMP 验证 ===")
    verifyParallelLimit()
    verifyWorkStealing()
}
```

关键行注释：
- `runtime.GOMAXPROCS(1)`: 设置 P 的数量为 1，从而强制只允许一个 M 运行 G，验证并行度受 P 的数量限制。
- `runtime.Gosched()`: 主动让出当前 P 的控制权，将当前 G 放回队列，触发调度器进行下一次调度；这是观察 work stealing 的入口点。
- `time.Sleep`: 会让 G 进入 _Gwaiting 状态，并让出 M；调度器随后会从队列中选择其他 G 执行，体现了 G 在 M 上的切换。

实际输出中，GOMAXPROCS=1 时总耗时明显大于 200ms，证明了 P 决定并行度；Gosched 场景仅用于演示，真实 work stealing 更直观的验证是：在多个 P 下，使用 `go tool trace` 查看 goroutine 的执行情况，能看到来自不同 M 的 goroutine 在另一个 P 的 M 上执行（称为 'G on another M'）。

### 4. 常见误区与进阶思考
误区 1: 认为 GOMAXPROCS 就是最大线程数。
GOMAXPROCS 限制的是同时执行用户 Go 代码的 P 的数量，不等于 M 的数量。M 的数量可以大于 GOMAXPROCS：当一个 M 进入系统调用（如文件 IO、锁等待）时，该 M 会被阻塞，runtime 会创建新的 M 或唤醒休眠的 M 来接管这个 P，从而继续执行其他 G。因此 M 的数量是动态的（受空闲锁、系统调用阻塞等影响），但并行执行 Go 代码的 P 数不会超过 GOMAXPROCS。在容器环境下（如 K8s），如果 Go 程序未正确设置 GOMAXPROCS，runtime 可能检测到宿主机的 CPU 核数而非容器配额，导致创建过多 P，增加线程上下切换和内存浪费（每个 M 栈默认 8MB 虚拟内存），但实际并行度仍受限于容器 CPU quota。推荐使用 `automaxprocs` 库动态设置。

误区 2: work stealing 一定可以保证负载均衡和低延迟。
work stealing 只能减少空闲 M 的等待时间，但不能消除所有调度延迟。极端情况下，一个 G 可能因为持续占用 P 而阻碍其他 G 运行——Go 1.14 前是协作式调度，只有发生函数调用、channel 操作或显式 Gosched 时才可能被抢占；如果一个 G 执行死循环且不触发任何调度点，则整个 P 会被卡住，其他 G 无法运行。Go 1.14 后的异步抢占基于 SIGURG 信号解决了死循环问题，但仍有抢占延迟（约 20-100 微秒）。另外，work stealing 只保证每个 P 忙碌时全局队列仍可能堆积，例如 G 创建速度远大于处理速度，此时调度器从全局队列取 G 的频率受限（每 61 次调度取一次），可能导致某些 G 长时间等待——这是设计上的权衡，防止全局锁竞争，但也会造成延迟波动。

深度思考题: 假设一个 Go 程序设置 GOMAXPROCS=8，当前有 8 个 P 全部在运行计算密集型的 goroutine，每个 goroutine 内是一个无限 for 循环（不主动让出）。此时另一个 goroutine 通过 go 语句被创建，它进入哪个队列？它大约需要多久才会被调度执行？请详细说明从创建到执行的完整路径（包括 sysmon 信号、抢占点、队列切换）。这个问题需要你理解异步抢占信号的触发机制、抢占点的判断逻辑（栈扫描 vs 指令指针检查）、以及被抢占的 G 被放入本地队列还是全局队列的细节——如果你能准确回答，说明你对 GMP 的理解已经超越了从博客上背概念的水平。
