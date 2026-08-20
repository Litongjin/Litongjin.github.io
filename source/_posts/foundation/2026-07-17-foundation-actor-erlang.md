---
title: "每日基础技术总结 · 2026-07-17 · Actor 模型与 Erlang 的调度"
date: 2026-07-17 08:00:00
categories: [技术分享]
tags: ["技术分享", "编程语言底层"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-17 · Actor 模型与 Erlang 的调度

## 📚 今日主题

> **Actor 模型与 Erlang 的调度**（编程语言底层）

### 1. 核心概念速览
Actor 模型是一种基于消息传递的并发计算抽象，其本质是将一切运行时实体视为独立的 Actor，每个 Actor 拥有唯一的地址、私有的状态和邮箱（Mailbox），并且仅通过异步消息进行交互。它解决的问题是共享内存并发模型下的数据竞争、锁顺序死锁和状态同步复杂性，机制是：Actor 之间的状态完全隔离，消息传递是唯一的通信方式，每个 Actor 按序处理其邮箱中的消息，且消息不可变（或至少在语义上不可变）。在计算机体系中的位置：它位于编程语言并发模型层，与 CSP（Communicating Sequential Processes）、STM（Software Transactional Memory）等并列，是 Erlang/OTP、Akka、Dapr 等系统的理论基石。专业工程师必须掌握它，因为它是构建高可用、可弹性扩展、容错分布式系统的核心范式之一，尤其是 Erlang/OTP 的监督树、软实时调度等机制直接依赖 Actor 模型，理解其底层调度能帮助你在设计后端服务时正确选择并发抽象，避免将线程锁的思维硬套到消息驱动系统上。

### 2. 底层原理剖析
底层原理分为三个层次：1）Actor 的运行时抽象：每个 Actor 拥有一个轻量级进程（Erlang 中称为 process，非 OS 线程），一个 mailbox 用于缓存消息，以及一个用于执行消息处理的入口函数（loop）。Actor 的状态只存在于其私有内存中，其他 Actor 无法直接读写。2）调度机制：Erlang 的 VM（BEAM）为每个调度器（Scheduler，通常绑定一个 OS 线程）维护一个运行队列，队列中的元素是轻量级进程。调度器按时间片轮转（reduction-based scheduling）执行每个进程。每个进程每次被调度时只消耗一定数量的 reductions（约 4000 次函数调用或指令执行），时间片用尽后被挂起，放回队列尾部。进程的抢占是协作式的：在 IO、receive、定时器等待等操作时主动让出调度。由于进程数量远大于 OS 线程，这种用户级线程（轻量级进程）的创建和上下文切换成本极低（微秒级）。3）消息传递与邮箱：send 操作将消息副本（或指向不可变 term 的引用，语义上为值传递）追加到目标 Actor 的 mailbox 尾部。receive 操作从 mailbox 中按模式匹配选择最早匹配的消息，并移出队列。调度器在进程执行 receive 时，若 mailbox 为空则挂起进程，并记录其被哪些消息模式匹配；当消息到达时，调度器唤醒匹配的进程。对比前端已有概念：Java 的接口是编译期契约，TS 的接口是结构类型约束，它们都是静态的、定义“能做什么”的类型系统概念；而 Actor 模型是运行时并发抽象，定义“如何并行与通信”。前端中与 Actor 类似的是 Web Worker 加 postMessage 模型：Worker 间无共享状态，通过消息传递通信，但 Worker 是重量级线程（OS 线程），且没有内置的 mailbox 选择机制；另外，Redux 的 action/reducer 也有点像 Actor 消息处理，但 Redux 是单线程同步的，没有调度器。

### 3. 基础代码与实战验证
```text
以下用 Erlang 写一个极简 Actor 计数器，展示 Actor 的循环接收与状态隔离。-module(actor_counter).-export([start/0, increment/1, get/1]).

start() ->
    %% 创建一个轻量级进程，入口为 loop/1，初始状态为 0。
    %% spawn 返回一个 Pid（进程标识符），这是该 Actor 的唯一地址。
    spawn(fun() -> loop(0) end).

increment(Pid) ->
    %% 向 Pid 发送消息 {increment, Self}，Self 用于回执。
    %% 该操作是异步的，消息被放入 Pid 的 mailbox 后立即返回。
    Pid ! {increment, self()},
    receive
        ok -> ok
    after 1000 -> timeout
    end.

get(Pid) ->
    Pid ! {get, self()},
    receive
        {value, V} -> V
    after 1000 -> timeout
    end.

loop(Count) ->
    receive
        {increment, From} ->
            %% 处理 increment 消息：更新状态为 Count+1，
            %% 然后通过 loop(Count+1) 进入下一次循环，
            %% 继续处理 mailbox 中的下一条消息。
            %% 注意：状态 Count 是函数的参数，存在于该进程的栈中，
            %% 其他进程无法直接访问。
            From ! ok,
            loop(Count + 1);
        {get, From} ->
            From ! {value, Count},
            loop(Count)
    end.

%% 执行：C = actor_counter:start(),
%% actor_counter:increment(C), actor_counter:increment(C),
%% actor_counter:get(C). 返回 2。

关键机制注释：1) spawn 创建的不是 OS 线程，而是 BEAM 上的轻量级进程，内存占用约 1KB，调度器会为其分配 reductions。2) receive 在 mailbox 无匹配消息时阻塞，但不会占用 CPU，调度器会将进程移入等待队列，直到消息到达才重新激活。3) 每次 loop 递归调用都不是尾递归优化的普通递归，而是尾递归，编译器会复用当前栈帧，不会增长栈。
```

### 4. 常见误区与进阶思考
误区 1：认为 Actor 之间消息传递是引用传递，因此可以用消息传递来避免拷贝开销。实际上，Erlang 中的消息语义是值传递，消息中的 term 会被复制到目标进程的堆中（除非是大型二进制在特定优化下可能引用计数共享，但语义仍不可变）。如果你在消息中放入一个大列表，拷贝开销是不可避免的。这要求你设计消息时尽量传递轻量级数据或引用（如 ETS 表、持久术语标识符）。误区 2：认为 Actor 模型等同于进程/线程，从而把 Actor 的创建当作重量级操作，过度复用 Actor 导致状态瓶颈。事实上，Erlang 的 Actor 是极轻量的（约 1KB 内存，创建约几微秒），应该按照“一个并发任务/一个状态所有者”来建模，而不是像线程池那样复用。过度复用会引入不必要的消息转发和顺序化，降低并发度。

思考题：在 Erlang 中，如果两个 Actor 同时向同一个 Actor 发送消息，那么接收 Actor 的 mailbox 中的消息顺序是否一定是发送的时间顺序？如果是，如何保证？如果不是，为什么？请从调度器将消息追加到 mailbox 的原子性以及网络通信（分布式 Erlang）中消息的到达顺序这两个层面回答，并解释在分布式环境下，消息顺序的确定性对一致性算法（如 Raft）的影响。
