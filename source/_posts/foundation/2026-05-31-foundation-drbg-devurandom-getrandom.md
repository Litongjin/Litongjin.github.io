---
title: "每日基础技术总结 · 2026-05-31 · 密码学随机数与 DRBG 的熵池：/dev/urandom 与 getrandom"
date: 2026-05-31 08:00:00
categories: [技术分享]
tags: ["技术分享", "安全基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-05-31 · 密码学随机数与 DRBG 的熵池：/dev/urandom 与 getrandom

## 📚 今日主题

> **密码学随机数与 DRBG 的熵池：/dev/urandom 与 getrandom**（安全基础）

### 1. 核心概念速览
密码学随机数与 DRBG 的熵池：/dev/urandom 与 getrandom

核心概念速览：
- 定义：密码学随机数是满足不可预测性、均匀分布等安全要求的随机字节流。熵池（Entropy Pool）是内核中收集物理噪声源的缓冲区；DRBG（Deterministic Random Bit Generator）是基于种子产生确定序列的算法，但种子来自熵池。
- 本质：熵池解决“从哪里获得真随机性”，DRBG 解决“如何将少量随机种子扩展为大量安全随机数”。/dev/urandom 和 getrandom 是用户态访问同一内核 CSPRNG 的两个接口，而非两种随机源。
- 解决的问题：用户进程无法直接接触硬件熵源，内核负责统一采集、混合、估算熵量，并提供安全导出接口。
- 位置：属于操作系统安全核心，位于内核随机子系统。所有 TLS 密钥、会话令牌、密码盐、钱包私钥都依赖它。
- 为何必须掌握：前端开发常用 Math.random() 生成非安全随机数，但后端与 AI 系统的加密逻辑一旦误用不安全的随机源，整个安全体系崩溃。理解熵池与 DRBG 才能正确选择接口，并理解为何某些接口会阻塞。

### 2. 底层原理剖析
底层原理剖析：

1. 熵源采集：内核从硬件 RNG（如 RDSEED）、中断时间戳、设备噪声、网络包时序等获取原始比特流。这些样本通过混合函数进入熵池。

2. 熵混合与估计：Linux 使用 ChaCha20 作为混合原语，将新样本与当前状态异或并置换。内核维护一个“熵估算值”，用于衡量池中未知比特量，防止攻击者故意提供可控噪声来污染随机性。

3. DRBG 生成：当用户请求随机数时，内核从熵池提取种子，初始化一个 DRBG 实例。Linux 的 DRBG 采用 ChaCha20 算法，种子为 256 位。之后每次输出随机字节时，DRBG 更新内部状态（计数器、密钥），并生成输出。整个过程类似一个密钥流生成器。

4. 接口行为：
   - /dev/urandom：字符设备，读取时永不阻塞。在 Linux 4.8+ 中，它与 /dev/random 共享同一 CSPRNG，因此不存在“熵耗尽导致安全降级”的问题。
   - getrandom()：系统调用，默认从 urandom 源取数；若指定 GRND_RANDOM 则读取 /dev/random 的传统阻塞逻辑（通常不推荐）。关键区别在于，getrandom 在启动早期（CSPRNG 未初始化）会阻塞，而 /dev/urandom 此时也会阻塞（早期实现可能不阻塞，但现代内核会）。

5. 对比前端概念：Java 的接口是运行时多态契约，任何实现类都必须遵循；TypeScript 的接口只存在于编译期，用于静态类型检查，运行时被擦除。同理，/dev/urandom 是文件系统层的接口，对应用表现为一个文件；getrandom 是系统调用层的接口，直接进入内核。二者底层都汇聚到同一个 random 核心，但抽象层级不同，语义也有差异（例如是否经过 VFS 层、是否需要打开文件描述符）。这种“不同层提供相似接口”的设计与 Java/TS 接口的差异有异曲同工之处。

### 3. 基础代码与实战验证
```text
基础代码与实战验证（Python，Linux 环境）：

    import os
    import ctypes

    # 1. os.urandom：Python 高层封装，内部优先使用 getrandom(2)，
    #    不可用时回退到 /dev/urandom。底层都是内核 CSPRNG。
    r = os.urandom(16)
    print('os.urandom:', r.hex())

    # 2. 直接读取 /dev/urandom：以文件方式访问，
    #    会经过 VFS 层，但最终仍调用内核 random 模块。
    with open('/dev/urandom', 'rb') as f:
        r2 = f.read(16)
    print('/dev/urandom:', r2.hex())

    # 3. 通过 ctypes 调用 getrandom 系统调用，flags=1 (GRND_NONBLOCK)。
    #    如果熵池尚未初始化，会抛出 BlockingIOError。
    libc = ctypes.CDLL(None)
    buf = ctypes.create_string_buffer(16)
    ret = libc.getrandom(buf, 16, 1)  # 1 即 GRND_NONBLOCK
    if ret == -1:
        err = ctypes.get_errno()
        raise OSError(err, os.strerror(err))
    print('getrandom:', buf.raw.hex())

关键说明：
- os.urandom 与读取 /dev/urandom 在底层一致，但 getrandom 在启动早期会等待初始熵就绪，而 /dev/urandom 在旧内核上可能不等待。
- 上述 ctypes 调用仅用于演示系统调用接口，生产代码应使用 os.urandom 或标准库中的安全随机数 API。
```

### 4. 常见误区与进阶思考
常见误区与进阶思考：

1. 误区：认为 /dev/urandom 一定比 getrandom 不安全。实际上，现代 Linux 中 urandom 与 getrandom（不带 GRND_RANDOM）使用同一个 CSPRNG，输出质量相同。urandom 仅在极早启动阶段（CSPRNG 未初始化）会阻塞，之后的非阻塞不降低安全性。

2. 误区：把“熵池耗尽”理解为随机性消失。在 Linux 4.8+ 的 ChaCha20 DRBG 设计中，熵池用于播种 DRBG，播种后 DRBG 的秘密状态足以持续产生不可预测输出。即使熵估算值降为零，只要内部状态未泄露，输出仍然安全。因此 getrandom 不需要在每次读取时等待新熵。

3. 深度思考题：为什么 getrandom(2) 在内核启动早期会阻塞，而一旦初始化完成后，即使熵池的估算熵为零也不再阻塞？请从 DRBG 的种子与状态演进角度解释阻塞与不阻塞的边界条件。回答要点：初始化前没有可用的秘密种子，任何输出都可被预测；初始化后 DRBG 已有 256 位秘密种子，输出取决于状态，攻击者无法从输出反推状态，因此无需持续补充熵。
