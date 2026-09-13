---
title: "每日基础技术总结 · 2026-09-14 · Docker 基础概念与镜像原理"
date: 2026-09-14 07:02:36
categories: [技术分享]
tags: ["技术分享", "后端基础（Node.js / Java / Python）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-14 · Docker 基础概念与镜像原理

## 📚 今日主题

> **Docker 基础概念与镜像原理**（后端基础（Node.js / Java / Python））

### 1. 核心概念速览
Docker 是一种基于 Linux 内核虚拟化技术的容器运行时，其本质是用户态进程隔离与资源控制，而非虚拟机。它利用 namespaces（命名空间）为进程提供独立的视图（PID、Network、Mount、UTS、IPC、User），利用 cgroups（控制组）限制资源使用（CPU、内存、IO），并通过 overlayfs 等联合文件系统实现分层镜像。Docker 解决的核心问题是：将应用及其完整运行环境打包为可移植的镜像，统一开发、测试、生产环境的不一致性，同时提供粒度适中的隔离，使同一宿主机可运行多个互不干扰的进程。在整个计算机体系中，Docker 位于操作系统与应用程序之间，属于操作系统级虚拟化层；在 AI 体系中，它是模型服务、训练任务环境复现的基础设施。专业工程师必须掌握其镜像分层、容器生命周期和网络模型，才能正确设计 CI/CD、微服务部署和资源隔离方案，否则只能停留在使用命令的表面层。

### 2. 底层原理剖析
Docker 镜像由只读层（layer）堆叠而成，每个层对应 Dockerfile 中的一条指令。写时复制（Copy-on-Write）机制保证容器写入只发生在最上层的可写层。底层原理：容器启动时，Docker 从镜像层堆叠中挂载一个联合文件系统（UnionFS），例如 overlay2。overlay2 将 lowerdir（镜像层）和 upperdir（容器可写层）合并为一个 mount point。读取文件时，若 upperdir 不存在则从 lowerdir 读取；写入时，若文件存在于 lowerdir 则先复制到 upperdir 再修改（COW）。删除文件时，在 upperdir 创建 whiteout 文件掩盖 lowerdir 的对应项。进程隔离依赖 Linux namespaces：clone() 系统调用时可指定 CLONE_NEWPID、CLONE_NEWNET、CLONE_NEWNS 等标志，使子进程获得独立的 PID、网络栈、挂载点等。资源限制依赖 cgroups：将进程放入 cgroup 后，通过写 /sys/fs/cgroup/cpu/.../cpu.max 等接口限制资源。Docker 通过网络命名空间默认创建 bridge 网络，通过 veth pair 将容器接口连接到 docker0 网桥，再通过 iptables NAT 实现外网访问。与前端对比：Docker 镜像层的分层类似前端模块化打包中的增量缓存——webpack 中不变的依赖层可被缓存，改动仅影响上层；但这种类比只描述构建效率，底层本质是文件系统层叠加。更进一步，容器对进程的隔离可类比浏览器的 origin 隔离：不同 origin 的 JS 上下文互不可见，但同一浏览器进程仍共享内核；同理容器共享宿主内核，比虚拟机更轻量，但隔离边界是内核 API，因此存在同源中毒的风险。这与 Java/TS 的接口无关，接口是编译期约束，Docker 是运行期内核机制，不能混淆。

### 3. 基础代码与实战验证
```text
# 极简 Dockerfile 验证分层与容器隔离机制

FROM alpine:latest
# 每条指令产生一个只读镜像层。下面两条 RUN 各自产生一层。
RUN echo layer1 > /data/file1
# 容器内 PID 1 是第一个进程，验证 PID namespace 隔离。
CMD ["sh", "-c", "cat /data/file1; echo 'PID in container:'; echo $$; echo 'PID on host:'; ps -p $$ -o pid="]

# 构建并运行：
# docker build -t test-layer .
# docker run --rm test-layer

# 底层运作注释：
# 1. FROM 拉取 alpine 基础镜像，其包含 rootfs 和包管理器，全部位于 lowerdir。
# 2. RUN echo layer1 > /data/file1 创建新层，该层只记录相对上一层的变更（新增文件内容）。
# 3. CMD 指定默认进程。docker run 时，Docker daemon 调用 runc 创建容器进程。
#    runc 使用 clone() 创建新进程，同时指定 CLONE_NEWPID 等标志，使容器内 PID 1 与宿主 PID 隔离。
# 4. 容器运行时，overlayfs 将镜像只读层叠加，并挂载一个临时可写层用于容器内写操作。
#    /data/file1 实际在 lowerdir 中，读取时从 lowerdir 获取，写入时触发 copy-up。
# 5. 容器进程看到的是独立的文件系统视图、进程表、网络栈，但内核与宿主机共享。

# 验证写时复制：在容器内修改文件后，镜像不受影响：
# docker run --rm test-layer sh -c "echo modified > /data/file1; cat /data/file1"
# docker run --rm test-layer cat /data/file1  # 仍输出 layer1，证明镜像层未被修改

# 验证 PID 隔离：进入容器执行 ps 看不到宿主进程；宿主执行 ps 能看到容器进程但 PID 不同。
```

### 4. 常见误区与进阶思考
误区一：将 Docker 容器当作轻量虚拟机。虚拟机通过 Hypervisor 虚拟化硬件，每个 VM 有独立内核；容器共享宿主机内核，隔离基于 namespaces/cgroups，不是恒等隔离。因此容器内 uname 显示的是宿主内核版本，且内核漏洞可能影响所有容器。专业工程师不应假设容器内的 root 与宿主 root 同等安全，必须配置 user namespace remap 和 drop capabilities。误区二：认为镜像层数越多镜像越小。实际上每个只读层都会增加联合挂载的元数据开销和构建缓存失效概率，且层内重复文件即使后续删除也会在较低层保留，最终导致镜像体积膨胀。正确做法是合并相关 RUN 指令，但也要权衡缓存利用。

思考题：如果两个容器共享同一个基础镜像层，其中一个容器在 /etc/hosts 写入内容，另一个容器能看到吗？为什么？请从 overlayfs 层挂载和 COW 的 physical 位置（根文件系统挂载点）角度分析，写出答案中涉及的内核对象名称。
