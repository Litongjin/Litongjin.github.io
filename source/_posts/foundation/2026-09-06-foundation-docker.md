---
title: "每日基础技术总结 · 2026-09-06 · Docker 基础概念与镜像原理"
date: 2026-09-06 07:01:43
categories: [技术分享]
tags: ["技术分享", "后端基础（Node.js / Java / Python）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-06 · Docker 基础概念与镜像原理

## 📚 今日主题

> **Docker 基础概念与镜像原理**（后端基础（Node.js / Java / Python））

### 1. 核心概念速览
Docker 是容器化生态的核心运行时引擎，其本质是在 Linux 内核层实现进程级别隔离与分发打包的系统。它通过镜像（Image）将应用及其依赖、配置、环境变量统一固化，再实例化为容器（Container）运行。Docker 解决的核心问题是环境一致性与交付标准化，使软件在开发、测试、生产全链路中消除环境差异。
底层机制依赖 Linux 内核能力：namespace 提供进程、网络、挂载点等视图的隔离，cgroups 提供 CPU/内存/IO 资源限制，OverlayFS 等 union filesystem 实现镜像分层与写时复制（Copy-on-Write）。Docker 本身不包含虚拟化内核，仅负责用户态编排与调用。
在整个计算机/AI 体系位置中，Docker 属于云原生基础设施层，位于操作系统与业务应用之间。AI 模型推理、训练环境的 GPU 驱动、CUDA、Python 依赖等复杂环境，最终都通过容器进行可复现交付。专业工程师必须掌握其镜像分层、容器生命周期、网络与存储模型，否则无法理解线上镜像体积膨胀、启动延迟、安全漏洞根因。

### 2. 底层原理剖析
镜像的本质：一个只读模板，由多个只读层（Layer）按顺序叠加而成。每一层不是文件系统的完整快照，而是该指令执行前后文件系统变化的 Diff 集合。
容器的本质：进程。启动容器时在镜像顶层挂载一个可写层，所有写操作写入该层。读取文件时从顶层向下查找，命中即返回；写入低层存在的文件时，触发 Copy-up 机制，将该文件从低层复制到可写层再修改，底层保持不变。删除低层文件时，不真正删除原数据，而是在可写层创建一个 whiteout 文件遮蔽它。
隔离机制：每个容器创建独立的 PID、NET、MNT、UTS、IPC、USER namespace，使进程只能看到 namespace 内资源。cgroups 限制容器可用的 CPU、内存、磁盘 IO。两者共同构成容器与传统虚拟机的根本区别——共享宿主内核。
与前端知识体系的对比：镜像分层与 JavaScript 原型链在机制上高度同构。镜像的底层只读层相当于原型对象，容器的可写层相当于实例的自身属性；读操作沿层向上查找，写操作创建自身遮蔽层，不修改原型。不同镜像共享的底层 layer 仅存储一份，类似模块共享的公共依赖项，通过 digest 进行内容寻址。
镜像构建时，Dockerfile 的每个 RUN / COPY / ADD 指令生成一个新层，构建器对各层缓存复用。若中间层缓存未失效，则不再执行指令。最终镜像的 manifest 记录层顺序与 digest，拉取时按层并行下载。

### 3. 基础代码与实战验证
```text
以下为极简验证原理的代码和命令，注释说明底层机制。

# Dockerfile
FROM alpine:latest        # 基础镜像层，提供完整根文件系统骨架
RUN echo "base-file" > /tmp/base.txt   # 该指令产生一个新层，这一层的 diff 是“/tmp/base.txt 被创建”
COPY --chown=0:0 ./app.py /app/app.py # 新层 diff：新增 /app/app.py
CMD ["sleep", "infinity"]

# 构建与验证命令

docker build -t demo:latest .
# 构建时逐条执行 Dockerfile，并将每个指令产生的文件系统变更记录为独立 layer

docker history demo:latest
# 显示每一层对应的指令和大小。注意 SIZE 不是最终镜像占用，而是相对于之前层的 diff 增量

docker inspect demo:latest --format '{{join .RootFS.Layers "\n"}}'
# 输出每一层的内容寻址摘要（sha256 digest），证明镜像由独立层叠加而成

docker run -d --name demo_c demo:latest
# 容器进程启动时，Docker 在镜像顶层挂载一个可写层，可写层初始为空

docker diff demo_c
# 对比容器文件系统与镜像文件系统，显示可写层中的变更：A 新增 / C 修改 / D 删除

docker exec demo_c sh -c 'echo overwritten > /tmp/base.txt'
docker diff demo_c
# 此时 /tmp/base.txt 显示为 C。底层镜像中的原始文件未被修改；OverlayFS 执行 copy-up，将底层文件复制到可写层并写入新内容。

docker exec demo_c rm /tmp/base.txt
docker diff demo_c
# 删除操作在可写层写入 whiteout 文件遮蔽底层文件，底层原文件仍在镜像中，OS 中计算空间也仍被镜像层占用。
```

### 4. 常见误区与进阶思考
误区一：将 Docker 容器视为轻量级虚拟机。容器内没有独立内核，所有容器共享宿主机内核（Windows/macOS 靠 Linux VM 中转，但本质不变）。隔离由 namespace 提供视图隔离，cgroups 提供资源限制，并非硬件级虚拟化安全边界。内核漏洞一旦被容器内进程利用，可直接影响整个宿主机。运维和隔离策略不能按虚拟机模型设计。
误区二：认为镜像大小是各层大小之和，且删除低层文件能减小镜像体积。镜像层是 immutable 的 diff，删除文件只是在可写层写入 whiteout 遮蔽，低层数据仍被镜像 digest 引用。如果直接 commit 一个新镜像，这个 whiteout 会成为新镜像独立的一层，而低层数据若没有其他镜像共享，仍被完整保留，最终镜像体积不降反增。只有通过全量重构镜像（重新 COPY，而不是基于 commit 或 dockerfile 在旧基础上删除）才能使数据真正脱离交付内容。
思考题：在容器内删除一个存在于镜像低层的大文件后，容器内 df -h 和 du -sh 为什么可能无法反映真正的可回收空间？请从 OverlayFS 的 whiteout 与 copy-up 机制分析，并设计一种可以不重新构建整个镜像，但能显著减少存储占用和镜像分发体积的实操方案。
