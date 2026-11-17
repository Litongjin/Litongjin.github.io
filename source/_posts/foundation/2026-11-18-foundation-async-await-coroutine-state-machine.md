---
title: "每日基础技术总结 · 2026-11-18 · async/await 背后的协程状态机"
date: 2026-11-18 07:02:03
categories: [技术分享]
tags: ["技术分享", "Python 工程化"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-18 · async/await 背后的协程状态机

## 📚 今日主题

> **async/await 背后的协程状态机**（Python 工程化）

### 1. 核心概念速览
在 CPython 中，async/await 是显式协程状态机：async def 编译为协程函数，调用返回 coroutine 对象；await 是可挂起边界，通过 __await__ 把当前帧从 yield 处冻结，并把待完成的 awaitable 交给 Task/Future/EventLoop。它解决的是单线程下用线性代码组织 I/O 并发，而不是自动获得 OS 并行。机制本质是 cooperative scheduling：协程主动让出，事件循环决定何时 send/throw 恢复。它位于 OS 线程、IO 复用、语言运行时、异步调度器之间；专业工程师必须掌握，因为 await、Task 引用、取消传播、背压和事件循环饥饿都直接决定后端与 AI 推理服务的吞吐、尾延迟和资源回收正确性。

### 2. 底层原理剖析
核心状态：
1. coroutine object：created / running / suspended / closing / closed；局部变量、执行点、异常状态保存在 PyFrameObject/CodeObject 中。
2. Future/Task：pending / done / cancelled；Task 是 EventLoop 可调度单元，负责驱动 coroutine 的 send 和 throw。
3. EventLoop：ready queue + I/O selector + timers；它不是 OS 调度器，只是用户态任务驱动器。

状态机流程伪代码：
loop = get_running_loop()
coro = async def f(): ...   # created
fut = loop.create_task(coro)
loop.run_forever()

Task.__step():
  try:
    yielded = coro.send(value_or_None)
  except StopIteration as e:
    fut.set_result(e.value)
    fut.mark_done()
  except CancelledError:
    fut.cancel()
  else:
    yielded.add_done_callback(Task.__wakeup)

Task.__wakeup():
  loop.call_soon(Task.__step)

Future.set_result(v):
  if not done:
    done = True
    for cb in callbacks: loop.call_soon(cb)

EventLoop._run_once():
  run ready callbacks
  poll selector for fd readiness
  fire timers
  enqueue new ready callbacks

await x 的底层语义：
  awaitable = get x
  iterator = awaitable.__await__()
  value = None
  while True:
    try: yielded = iterator.send(value)
    except StopIteration as e: return e.value
    # yielded 通常是 Future；协程帧在此冻结，控制权返回 Task/EventLoop
    value = yielded.result_or_wait()

与前端 async/await 对比：
1. JS/TS async function 返回 Promise，await 被编译为 Promise 微任务续体；Python async def 返回 coroutine object，await 需要 __await__ 协议。
2. JS 有 microtask queue 与 macrotask queue；Python asyncio 主要用 call_soon ready queue、selector poll、timer handle，没有原生 microtask 概念。
3. JS 取消依赖 AbortSignal；Python 取消通过 Task.cancel 向 coroutine 注入 CancelledError，并由 await 边界传播。
4. JS async function 自动封装 Promise；Python coroutine 不会自动进入事件循环，必须被 Task 包裹或被显式 send 驱动。
5. 相似点是：两者都是语言层协程，都是单线程协作式调度，都靠事件循环恢复执行点，而不是创建线程。

### 3. 基础代码与实战验证
```text
import asyncio

async def state_probe():
    # await 是显式挂起点：协程执行到这里会把内部 Future 从帧中 yield 出来。
    await asyncio.sleep(0.01)
    return 'done'

async def main():
    coro = state_probe()
    # 此处 coroutine 处于 created 状态；没有任何 OS 线程切换，也没有代码执行。
    print('coroutine not executed:', type(coro).__name__)

    task = asyncio.create_task(coro)
    # create_task 把 Task.__step 放入事件循环就绪队列；Task 开始驱动 coro.send。
    print('task done before await:', task.done())

    # await task 让 main 挂起。事件循环随后执行 Task.__step：
    # state_probe 在 await asyncio.sleep 处 yield Future；timer 完成后回调恢复 send。
    result = await task
    print('task done after await:', task.done())
    print('result:', result)

asyncio.run(main())

# 验证结论：
# 1. async def 调用不执行，只创建 coroutine 对象。
# 2. Task 是协程状态机与 EventLoop 之间的适配器。
# 3. await 不切换 OS 线程，只切换用户态执行点。
```

### 4. 常见误区与进阶思考
误区一：把 await 当成并发。顺序书写两个 await 会串行执行；真正并发来自 create_task 或 asyncio.gather 将多个协程同时交给事件循环。误区二：把 async 函数当成线程安全或 CPU 并行容器。事件循环单线程协作，任何同步 CPU、阻塞 IO、死锁锁都会导致整个 loop 饥饿；需要 run_in_executor、进程池或异步原生驱动。深度思考题：如果一个 await 表达式在 Future 已经 set_result 之后才执行，协程为什么可以不挂起、不进入事件循环而直接取得结果？这对事件循环公平性、递归深度和 CPU 饥饿意味着什么？
