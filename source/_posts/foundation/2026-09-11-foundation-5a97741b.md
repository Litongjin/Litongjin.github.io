---
title: "每日基础技术总结 · 2026-09-11 · 进程与线程的区别"
date: 2026-09-11 07:01:48
categories: [技术分享]
tags: ["技术分享", "后端基础（Node.js / Java / Python）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-11 · 进程与线程的区别

## 📚 今日主题

> **进程与线程的区别**（后端基础（Node.js / Java / Python））

### 1. 核心概念速览
进程是操作系统进行资源分配的基本单位，线程是操作系统进行CPU调度的基本单位。进程拥有独立的地址空间、文件描述符表、信号处理器等资源，线程共享所属进程的地址空间和大部分资源，但拥有独立的栈、寄存器上下文和程序计数器。进程解决的是资源隔离与多任务并行的问题，线程解决的是在共享资源下提高并发性和降低上下文切换开销的问题。在计算机体系结构中，进程位于操作系统内核之上，线程位于进程之内；AI框架（如TensorFlow、PyTorch）的并行训练既依赖多进程实现数据并行，也依赖多线程实现算子内的并行计算。专业工程师必须掌握二者差异，否则无法正确设计并发模型、诊断资源竞争或优化系统吞吐。

### 2. 底层原理剖析
底层机制由内核调度器和内存管理单元（MMU）共同实现。进程的创建（fork/clone）会复制或写时复制页表，切换进程需要切换CR3寄存器（更新MMU映射）、刷新TLB、切换内核栈，开销高；线程（如pthread_create或clone带CLONE_VM标志）只创建新的task_struct和栈，共享同一mm_struct，切换时无需切换地址空间，因此开销小。并发本质：CPU通过时间片轮转调度任务，每个任务在某一时刻只能在一个CPU核心上执行。进程间通过IPC（管道、共享内存、消息队列）通信，需内核介入或显式同步；线程间通过共享内存直接读写，但需原子操作或锁保证一致性。对比前端已有的『接口』概念：Java的接口是类型系统契约，TS的接口是编译期结构约束，二者都是静态的代码层抽象；而进程/线程是操作系统层的运行时实体抽象，接口解决的是代码解耦，进程/线程解决的是并发执行与资源管理，两者不在同一抽象层级。

### 3. 基础代码与实战验证
```text
// 验证进程与线程的地址空间隔离性（Linux/Unix环境，Node.js）
const { fork } = require('child_process');
const { Worker, isMainThread, parentPort } = require('worker_threads');

let globalVar = 42; // 进程级变量

// 子进程：fork会复制整个内存镜像，修改不会影响父进程
if (fork()) { // 父进程分支
  setTimeout(() => {
    console.log('父进程看到globalVar =', globalVar); // 仍为42
  }, 100);
} else {
  globalVar = 0; // 子进程修改自己的私有副本
  process.exit();
}

// 验证线程共享地址空间：worker_threads共享内存（需通过SharedArrayBuffer）
if (isMainThread) {
  const shared = new SharedArrayBuffer(4);
  const arr = new Int32Array(shared);
  arr[0] = 42;
  const worker = new Worker(`
    const { parentPort } = require('worker_threads');
    const arr = new Int32Array(require('worker_threads').workerData);
    arr[0] = 0; // 直接修改共享内存
    parentPort.postMessage('done');
  `, { eval: true, workerData: shared });
  worker.on('message', () => {
    console.log('主线程看到arr[0] =', arr[0]); // 0，证明线程共享同一物理内存
  });
}
```

### 4. 常见误区与进阶思考
误区一：认为线程一定比进程性能好。实际上在多核CPU且需要大量内存隔离场景下，进程因并行度更高（避免锁竞争和伪共享）可能整体性能更优，比如Chromium的多进程架构。误区二：混淆『并发』与『并行』。多线程是并发（交错执行），多进程在多核上才是并行（同时执行）；单核CPU下线程的切换只是时间片交错，并无真正的并行。思考题：在Linux中，使用clone系统调用创建进程和线程时，如何通过CLONE_VM标志决定二者区别？如果仅指定CLONE_VM但不共享文件系统信息，那么得到的是进程还是线程？请从内核task_struct和mm_struct的关系说明本质。
