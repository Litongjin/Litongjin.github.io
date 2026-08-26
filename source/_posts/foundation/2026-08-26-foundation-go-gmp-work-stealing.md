---
title: "每日基础技术总结 · 2026-08-26 · Go 的 GMP 调度器与 work stealing"
date: 2026-08-26 08:00:00
categories: [技术分享]
tags: ["技术分享", "编程语言底层"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-26 · Go 的 GMP 调度器与 work stealing

## 📚 今日主题

> **Go 的 GMP 调度器与 work stealing**（编程语言底层）

### 1. 核心概念速览
GMP 是 Go 运行时用于调度 goroutine 的三级调度模型：G（Goroutine）表示一个用户态协程，携带栈、PC、SP 等执行上下文；M（Machine）表示操作系统线程，负责实际执行 G；P（Processor）表示调度上下文，持有本地可运行 G 队列并决定哪些 G 可以在 M 上运行。P 的数量由 GOMAXPROCS 控制，默认等于 CPU 逻辑核数。调度的本质是：将用户态协程（G）多路复用到数量有限的 OS 线程（M）上，并在 P 之间动态平衡负载。work stealing 是调度器在某个 P 的本地队列空闲时，从其他 P 的本地队列尾部窃取 G 的机制，目标是最大化 CPU 利用率、最小化线程空转。GMP 位于编程语言运行时层，介于操作系统线程调度与用户业务代码之间，是 Go 实现高并发、低阻塞的基石。专业工程师必须掌握它，因为并发程序的性能、延迟、锁竞争、调度饥饿等问题都根源于调度器行为，理解 GMP 才能写出可预测、可调优的系统，而不仅仅是『go func 很轻量』的肤浅认知。

### 2. 底层原理剖析
运行时结构：每个 P 维护一个本地可运行队列（LRQ），容量通常为 256；所有 P 共享一个全局可运行队列（GRQ）。M 通过 P 获取 G：M 必须绑定一个 P 才能执行 G，未绑定的 M 不参与调度。调度循环：M 从当前 P 的 LRQ 头部弹出一个 G 并执行；当 G 主动让出（channel 操作、time.Sleep、runtime.Gosched）或发生抢占时，M 将当前 G 状态保存并放回队列，再从 LRQ 获取下一个 G。若 LRQ 为空，M 首先从 GRQ 取一批 G（批量获取以摊薄全局锁成本）；若 GRQ 也为空，则触发 work stealing：随机选取其他 P，从其 LRQ 的尾部窃取一半 G（约 n/2），如果所有 P 都为空，则进入自旋或阻塞等待。work stealing 的算法核心：受害者 P 的尾部被窃取，而不是头部，因为头部可能正在被该 P 的当前 M 执行或即将执行，尾部窃取能减少对热数据（头部）的竞争。调度器事件类型：(1) 主动调度：G 调用 park/unpark（同步原语）进入等待，M 切换到其他 G；(2) 被动调度：系统调用阻塞或时钟中断触发抢占，Go 1.14 后引入基于信号的异步抢占，解决长时间计算导致的调度延迟；(3) 协作式调度：函数调用时会检查抢占标志。syscall 处理：当 G 执行阻塞系统调用时，M 会与 P 解绑，P 转交给其他空闲 M 或新建 M 继续执行其他 G，系统调用返回后 G 重新进入调度队列。handoff 机制确保阻塞不会浪费 P。与前端事件循环的对比：浏览器/Node 的 event loop 本质是单线程的，只有一个主线程反复执行『取任务-执行-取任务』，所有异步靠事件回调/微任务队列，CPU 密集型任务会阻塞整个循环；而 GMP 是多对多模型，多个 M（OS 线程）可以并行执行多个 G，P 类似『独立的微任务队列+执行线程』，但多个这样的队列在多个线程上并行运行，且队列之间可以动态负载均衡。前端开发者熟悉的 promise 只是任务状态的抽象，而 G 是真正的执行上下文，可以随时被切换（抢占），promise 则不能随意中断执行。

### 3. 基础代码与实战验证
```text
// 验证 GMP 的核心：work stealing 与 P 并发度
package main

import (
    "fmt"
    "runtime"
    "sync"
)

func main() {
    // 限制 P 数量为 2，使 work stealing 更易观察
    // GOMAXPROCS 决定运行时创建多少个 P，进而决定同时有多少 M 在并行执行 G
    runtime.GOMAXPROCS(2)

    var wg sync.WaitGroup

    // 创建 4 个 goroutine，远多于 P 数，触发队列与窃取
    for i := 0; i < 4; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            // 连续占用 CPU，模拟密集计算，避免主动让出
            // 此循环会触发调度器的时钟抢占（基于信号），但执行期间 M 不会被切换走
            sum := 0
            for j := 0; j < 1000000; j++ {
                sum += j
            }
            // 获取当前 goroutine 运行所在的 OS 线程 ID（M 的标识）
            // runtime.Stack 可获取 goroutine 信息，但为了简洁，仅输出 P 的并行能力
            fmt.Printf("goroutine %d 完成, sum=%d, 当前并发执行：查看可同时运行的 goroutine 数 <= GOMAXPROCS\n", id, sum)
        }(i)
    }

    // 主动让出主 goroutine，使调度器有机会分配 G 到其他 M
    // 这里主 goroutine 进入等待，调度器会将其 park，并运行其他 G
    runtime.Gosched()

    wg.Wait()
    // 真实场景中可通过 trace 工具看到：
    // 1. 初始时每个 P 的 LRQ 被分配部分 G；
    // 2. 当某 P 先完成其 LRQ 中的 G，会从另一个 P 的 LRQ 尾部窃取 G。
    // 验证方法：运行 go run -race，并开启 go tool trace 观察 ProcStart/Goroutine 事件。
}

// 伪代码描述 work stealing 过程：
// func schedule(mp *m) {
//     gp := pop(mp.p.localQ)          // 先取本地队列头部
//     if gp == nil {
//         gp = popGlobal()            // 全局队列为空？
//         if gp == nil {
//             gp = stealWork()         // 随机选 P，从尾部窃取一半 G
//         }
//     }
//     execute(gp)                      // 绑定 M 到 P 执行
// }
```

### 4. 常见误区与进阶思考
误区 1：认为 GOMAXPROCS 是『最大线程数』或『最大并发 goroutine 数』。实际上它限制的是 P 的数量，而 P 决定了同时可运行的 G 数量上限（因为每个 M 需绑定 P 才能执行 G）。M 的数量可能大于 P，因为阻塞系统调用会创建额外 M，但并行执行的 G 数永远不会超过 GOMAXPROCS。误区 2：认为 goroutine 是『无限轻量』且永远不阻塞线程。实际上每个 G 的初始栈只有 2KB，但可增长到 1GB；更重要的是，如果 G 执行了无法被抢占的 CGO 调用或进入阻塞系统调用，会占用 M，导致运行时创建新线程，极端情况下可能产生大量 M，造成资源压力。work stealing 也并非保证绝对公平：它偏向于优先执行本地队列，可能导致某些 G 长时间等待（但 Go 1.14 后的抢占缓解了这个问题）。
思考题：若一个 G 执行纯计算循环且从不主动让出，且没有函数调用（内联展开），Go 1.14 之前的调度器会发生什么？Go 1.14 之后如何解决？请结合信号处理与 P 的抢占标志位，说明为什么纯循环也能被中断，以及中断后 G 的状态被保存到何处。
