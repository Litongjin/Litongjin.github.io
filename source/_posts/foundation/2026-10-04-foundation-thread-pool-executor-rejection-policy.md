---
title: "每日基础技术总结 · 2026-10-04 · 线程池：ThreadPoolExecutor 参数与拒绝策略"
date: 2026-10-04 07:03:47
categories: [技术分享]
tags: ["技术分享", "Java 后端与 Spring 生态"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-04 · 线程池：ThreadPoolExecutor 参数与拒绝策略

## 📚 今日主题

> **线程池：ThreadPoolExecutor 参数与拒绝策略**（Java 后端与 Spring 生态）

### 1. 核心概念速览
ThreadPoolExecutor 是 Java java.util.concurrent 包中基于线程池的 ExecutorService 实现，其本质是一个有限状态机驱动的任务调度器：维护一个 workerCount 的原子状态，结合阻塞队列和饱和策略，决定每个提交的任务由哪个执行体在何时执行。它解决的问题是线程创建/销毁的高昂成本与系统资源边界之间的冲突，通过复用有界的一组工作线程，将任务提交与任务执行解耦，并允许通过参数精确控制并发度、排队深度和过载行为。在计算机体系中，线程池位于应用线程与操作系统线程之间，是一种用户态调度层，负责管理线程生命周期、任务队列和资源分配。后端工程师必须掌握它，因为服务端高并发场景下，线程池是最常见的任务执行容器，它的参数配置直接影响应用的 RT、吞吐量、CPU 占用和内存稳定性；同时，线程池的异常表现（线程饥饿、任务丢失、OOM）是生产故障的主要来源之一。只有从底层理解其参数与拒绝策略之间的关系，才能正确地配置容量、排查问题并预测系统极限。

### 2. 底层原理剖析
ThreadPoolExecutor 的工作流程可归纳为三条路径，由内层 AtomicInteger ctl 的低 29 位记录 workerCount、高 3 位记录 runState。当调用 execute(Runnable) 时，执行以下逻辑（用伪代码描述）：

1. 如果 workerCount < corePoolSize，则直接启动一个新的 Worker（本质是持有 Thread 的内部对象），调用 addWorker(command, true) 执行任务；
2. 否则，尝试将任务入队 workQueue.offer(command)。若入队成功，则需二次检查 runState（防止线程池已关闭），并确认 workerCount 不为 0（防止线程都被回收后任务无人执行）；
3. 若队列已满或入队失败，再次尝试 addWorker(command, false)。若当前 workerCount < maximumPoolSize，则创建新的非核心线程执行该任务；否则，调用 handler.rejectedExecution(command) 执行拒绝策略。

注意：非核心线程的创建不是发生在核心线程全部忙碌时，而是发生在队列已满且还有最大线程余量时。keepAliveTime 控制非核心线程在空闲时的存活时间；若调用 allowCoreThreadTimeOut(true)，核心线程同样受 keepAliveTime 管理。

状态转换：线程池运行状态包括 RUNNING、SHUTDOWN、STOP、TIDYING、TERMINATED。每次任务提交或线程退出都会使用 CAS 更新 ctl，保证并发正确性。

与前端已有概念的对比：前端单线程事件循环（Event Loop）天然避免了多线程竞争，但没有显式的并发调度；而 Node.js 中的 libuv 线程池（默认 4 个线程）是类似的后端资源池，但它是面向文件 I/O 和 DNS 这类阻塞操作，没有公开的拒绝策略。ThreadPoolExecutor 则是显式、可配置、可监控的线程复用机制，其抽象接口 ExecutorService 与 ThreadPoolExecutor 的关系更像 TS 中 interface 与 class implements 的关系，但 Java 接口在运行时还可通过动态代理等机制反射性地参与执行链，而 TS 接口在编译后完全消失——这提醒我们 JVM 层面对抽象的层次有真实的运行期形态，理解这种差异有助于建立跨语言后端体系。

### 3. 基础代码与实战验证
```text
import java.util.concurrent.*;

public class ThreadPoolDemo {
    public static void main(String[] args) throws InterruptedException {
        // 核心线程数=1，最大线程数=2，空闲存活时间=0ms（非核心线程立即回收）
        // 任务队列容量=1，拒绝策略=AbortPolicy（默认，直接抛异常）
        ThreadPoolExecutor tp = new ThreadPoolExecutor(
                1, 2, 0L, TimeUnit.MILLISECONDS,
                new LinkedBlockingQueue<>(1),
                Executors.defaultThreadFactory(),
                new ThreadPoolExecutor.AbortPolicy());

        for (int i = 1; i <= 4; i++) {
            final int taskNo = i;
            try {
                // execute 内部按“核心→入队→非核心→拒绝”顺序流转
                tp.execute(() -> {
                    System.out.println(Thread.currentThread().getName() + " 执行任务 " + taskNo);
                    try { Thread.sleep(100); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
                });
            } catch (RejectedExecutionException e) {
                System.out.println("任务 " + taskNo + " 被拒绝");
            }
        }
        // 提交顺序：任务1（核心线程）、任务2（入队）、任务3（创建非核心线程）、任务4（拒绝）
        tp.shutdown();
    }
}
```

### 4. 常见误区与进阶思考
误区一：认为 keepAliveTime 会回收所有空闲线程。默认情况下，keepAliveTime 只作用于非核心线程（即超出 corePoolSize 的部分）。要回收核心线程，必须显式调用 allowCoreThreadTimeOut(true)。在 Executors.newFixedThreadPool 中线程数固定为 corePoolSize，keepAliveTime 为 0 并没有意义，因为它不回收核心线程。

误区二：把 Executors.newCachedThreadPool 当作高性能万能池。newCachedThreadPool 使用 SynchronousQueue，该队列不持有任务，每个任务到达后需要立即有空闲线程接走，否则会创建新线程，导致线程数无上限地跟随并发量增长，最终可能资源耗尽。而 newFixedThreadPool 使用无界 LinkedBlockingQueue，当任务积压时只会排队而不会创建额外线程，容易造成内存 OOM。生产环境必须手动指定有界队列和明确的最大线程数及拒绝策略。

思考题：给定 ThreadPoolExecutor(corePoolSize=2, maximumPoolSize=4, workQueue 容量=10)，当前有 2 个核心线程都在执行任务，队列中已有 10 个任务等待。此时再提交第 13 个任务，会发生什么？请描述该任务是被执行、入队还是拒绝，并解释为什么。
