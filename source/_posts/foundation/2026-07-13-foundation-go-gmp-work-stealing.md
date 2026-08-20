---
title: "每日基础技术总结 · 2026-07-13 · Go 的 GMP 调度器与 work stealing"
date: 2026-07-13 08:00:00
categories: [技术分享]
tags: ["技术分享", "编程语言底层"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-13 · Go 的 GMP 调度器与 work stealing

## 📚 今日主题

> **Go 的 GMP 调度器与 work stealing**（编程语言底层）

### 1. 核心概念速览
Go的GMP调度器是运行时实现用户态并发调度的核心模型。G是Goroutine，M是操作系统线程，P是逻辑处理器（调度上下文）。调度器将G调度到M上执行，通过P限制并行度。本质是M:N用户态调度器，解决内核线程创建/切换开销高、多核负载不均衡等问题。它在操作系统线程之上、用户代码之下，构成Go并发执行的底座。专业工程师必须掌握，因为并发性能、阻塞、系统调用、GC暂停都与之直接关联。

### 2. 底层原理剖析
核心调度循环：每个P维护一个本地运行队列（runq，容量256），新G先放当前P的runnext（栈顶，LIFO）或本地队列尾部；满则放入全局队列。M要执行G必须先获取P。调度循环中：1）从P本地队列弹出一个G；2）本地空则从全局队列取（每60次调度取一次）；3）全局也空则执行work stealing：随机选择其他P，窃取一半G。若仍空，M将P放入空闲列表并休眠，等待信号。当G执行阻塞操作（如channel、系统调用），M与P解绑，P可以被其他空闲M接管，阻塞G进入等待队列，阻塞M休眠。抢占：runtime的sysmon每10ms检查运行中的G，若超过10ms未让出，则发起异步抢占（信号），强制G陷入调度。对比前端：JS事件循环是单线程，无并行，任务队列FIFO，无抢占；Go的GMP是多线程并行，每个P独立队列，支持work stealing，且由runtime抢占。这类似于Java的ForkJoinPool，但Go将调度与语言运行时深度整合。

### 3. 基础代码与实战验证
```text
以下为验证调度器行为的极简Go代码（字符串使用反引号避免转义）：
package main

import (
	"fmt"
	"runtime"
	"sync"
)

func main() {
	runtime.GOMAXPROCS(2) // 设置P数量为2，最多允许2个M并行执行G
	var wg sync.WaitGroup
	ch := make(chan int)

	// G1: 消费者。创建时被放入当前P的runnext，等待调度。
	wg.Add(1)
	go func() {
		defer wg.Done()
		for v := range ch {
			fmt.Println(`recv`, v)
		}
	}()

	// 主Goroutine向channel发送数据。发送时若缓冲区满，G会阻塞，
	// 触发调度器将当前M与P解绑，P让给其他M，从而消费者G被调度执行。
	for i := 0; i < 3; i++ {
		ch <- i
	}
	close(ch)
	wg.Wait()
	// 实际运行时，M会从P的本地队列取G执行，并在阻塞点切换G。
	// 若两个P都存在，一个P队列空时，会从另一个P窃取任务（work stealing）。
}


调度器主循环（伪代码）：
for {
	g := p.runnext // LIFO优先
	if g == nil { g = p.runq.pop() }
	if g == nil { g = globalQueue.pop() }
	if g == nil {
		for _, p2 := range allP {
			g = p2.runq.steal() // 窃取一半
			if g != nil { break }
		}
	}
	if g == nil { park() } else { execute(g) }
}
```

### 4. 常见误区与进阶思考
常见误区1：认为GOMAXPROCS设置的是CPU核心数，实际上它设置的是P的数量，进而限制并行执行的M数量（非阻塞时）。误区2：认为Goroutine数量越多越好，忽略P的本地队列容量和调度开销，导致性能下降。思考题：在GOMAXPROCS=1时，一个Goroutine执行无限循环，另一个Goroutine能否运行？为什么？
