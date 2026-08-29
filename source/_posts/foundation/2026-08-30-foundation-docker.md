---
title: "每日基础技术总结 · 2026-08-30 · Docker 基础概念与镜像原理"
date: 2026-08-30 06:57:23
categories: [技术分享]
tags: ["技术分享", "后端基础（Node.js / Java / Python）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-30 · Docker 基础概念与镜像原理

## 📚 今日主题

> **Docker 基础概念与镜像原理**（后端基础（Node.js / Java / Python））

### 1. 核心概念速览
Docker 是一种基于操作系统级虚拟化的容器引擎，本质是利用 Linux 内核的 namespace（命名空间）与 cgroups（控制组）将单个进程树隔离为独立的运行环境，并通过镜像（Image）作为只读的、分层存储的文件系统模板来实例化为容器（Container）。它解决的核心问题是软件交付中『环境一致性』与『依赖封装』：将应用及其运行时、库、配置全部打入镜像，从而抹平开发、测试、生产环境的差异。Docker 位于操作系统与应用程序之间的虚拟化层，是云原生、DevOps、微服务与 AI 模型部署体系中的基础设施。专业工程师必须掌握它，因为它不仅是部署工具，更是理解 Linux 隔离机制、存储驱动、网络协议栈和系统调用的直观入口；后端与 AI 工程师则依赖它作为运行环境的标准封装与交付单元。

### 2. 底层原理剖析
镜像原理的核心是分层与联合挂载。底层采用 OverlayFS 或类似 Union File System，将多个只读层叠加为一个统一视图。每个 Dockerfile 指令生成一个只读层，层的标识基于该层内容的哈希；构建时若某层及其父层缓存有效，则直接复用该层，不重复执行指令。容器启动时在内核中创建新的 namespace（PID、NET、MNT、UTS、IPC）并应用 cgroups 资源限制，同时联合挂载镜像的所有只读层，并在顶层挂载一个可写的临时层。当容器内写文件时，触发 copy-up 机制：将文件从下层复制到可写层后修改，下层保持不可变。

伪代码描述容器启动流程：
1. daemon 接收 run 请求
2. create container：根据镜像配置分配 rootfs，挂载 lowerdir=镜像层列表，upperdir=/var/lib/docker/overlay2/<id>/diff
3. 设置 namespace/env/mount point
4. 设置 cgroup 资源配额
5. 执行 CMD 指定的进程

与前端概念的对比：镜像相当于前端构建产物（如 dist 目录），但 dist 只包含静态文件，镜像则包含完整的操作系统文件系统视图；容器相当于运行该产物的进程，但拥有独立文件系统、网络栈和主机名；Dockerfile 的层缓存机制类似于 webpack 的持久化缓存（基于内容哈希），但 Docker 的缓存粒度是文件系统快照级，缓存命中可跳过整条指令。这与 Java 接口/TS 接口的『定义与实现分离』不同——Docker 不是语言层面的抽象，而是操作系统级的运行时装配，镜像和容器是『定义』与『实例化』的关系，约束来自文件系统层而非类型系统。

### 3. 基础代码与实战验证
```text
FROM node:18-slim
# 每个指令创建一个层，FROM 创建基础镜像层
WORKDIR /app
COPY package.json /app/
# 先复制依赖清单，只要该文件不变，后续 RUN 产生的层可复用（构建缓存）
RUN npm install
# 这一层将 npm install 生成 node_modules 提交为只读层
COPY . /app
CMD ["node", "server.js"]

# 构建与验证：
docker build -t demo .
docker history demo
# 输出每层的 ID、大小、创建指令，验证分层结构

# 验证写时复制：
docker run -it --rm demo sh
# 容器内创建文件 /tmp/new.txt，该写入发生在顶层可写层；
# 退出后容器销毁，镜像本身没有任何变化——可写层不会污染镜像。
```

### 4. 常见误区与进阶思考
1. 将容器等同于虚拟机：容器共享宿主机内核，没有独立 Kernel，隔离基于 namespace 而非硬件虚拟化；同一宿主机的内核漏洞可能被容器利用，安全边界弱于 VM。
2. 认为镜像是一个单一的大文件：镜像本质是分层文件系统，Dockerfile 每条指令都会生成新层，修改文件会追加层而不是替换旧层；如果指令顺序不合理或层数过多，会导致构建缓存频繁失效、镜像体积膨胀。

思考题：若宿主机已有镜像 A = base(ubuntu:20.04) + layerA，镜像 B = base(ubuntu:20.04) + layerB，请问两个镜像的 base 层磁盘占用是否翻倍？结合存储驱动的层共享机制解释实际占用量与生成容器的文件系统结构。
