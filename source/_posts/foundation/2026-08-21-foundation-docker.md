---
title: "每日基础技术总结 · 2026-08-21 · Docker 基础概念与镜像原理"
date: 2026-08-21 06:55:27
categories: [技术分享]
tags: ["技术分享", "后端基础（Node.js / Java / Python）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-21 · Docker 基础概念与镜像原理

## 📚 今日主题

> **Docker 基础概念与镜像原理**（后端基础（Node.js / Java / Python））

### 1. 核心概念速览
Docker 本质上是基于 Linux 内核原语的 OS 级虚拟化（容器化）实现，不是虚拟机：所有容器与宿主机共享同一个内核。隔离由 Namespaces（命名空间）提供，资源约束由 Cgroups（控制组）提供，文件系统视图由 UnionFS（OverlayFS）实现。Docker 镜像是一个只读、多层、内容寻址的不可变模板；容器则是在镜像之上叠加一个可写层后运行起来的进程实例，进程入口由镜像中的 CMD/ENTRYPOINT 定义。它解决的核心问题是环境一致性（构建一次，随处运行）、交付物标准化与依赖隔离，将软件交付从『源码 + 环境手工配置』转变为『不可变工件 + 标准运行时』。在计算机体系中的位置：位于操作系统内核与用户进程之间，是进程级打包与分发的基础设施，也是 Kubernetes 等编排系统的原子部署单元；在 AI 体系中，GPU 容器（nvidia-container-toolkit）、推理服务（Triton、vLLM）、MLOps 流水线全部默认以容器为交付格式。专业工程师必须掌握它，因为不理解镜像分层、运行时隔离与写时复制，就无法调试生产环境中的诡异问题，也无法设计镜像瘦身策略、缓存策略与安全的 CI/CD 流水线。

### 2. 底层原理剖析
底层运行机制按链路拆解如下。
1. 构建（docker build）：Dockerfile 的每条指令（FROM、WORKDIR、COPY、RUN、CMD）被 BuildKit 执行为一次文件系统变更，变更结果以层（layer）为单位不可变地提交。每层本质是执行该指令后整个根文件系统的一个快照，以 tar 流与元数据存储，并用 SHA256 摘要做内容寻址（digest）。层之间构成父子链，同 Git 提交对象一样：内容决定身份，父链决定顺序。
2. 存储（UnionFS/OverlayFS）：运行容器时，所有只读镜像层按顺序叠加为 OverlayFS 的 lowerdir，另建一个可写层作为 upperdir（容器层）。容器内看到的根文件系统是各层合并视图：同名文件按层序遮蔽（上层覆盖下层），同名目录则合并。
3. 运行（runc）：docker run 经 dockerd 与 containerd 最终由 runc 完成：创建独立的 Namespace 集合（PID、Network、Mount、UTS、IPC、User），使容器进程拥有独立的进程号空间、网络栈、挂载点与主机名；挂载 OverlayFS 后通过 pivot_root 将进程根目录切换到合并视图；随后以镜像中 CMD 指定的可执行文件作为容器内 PID 1 启动，同时建立 Cgroup 限制 CPU、内存与 I/O。
4. 写时复制（Copy-on-Write）：容器内首次修改某文件时，OverlayFS 将 lowerdir 中的该文件整体复制（copy-up）到 upperdir 再写入，之后读写均在可写层进行。因此同一镜像可同时支撑大量容器，互不干扰，镜像本身永远不变。
5. 删除即白障（whiteout）：容器内删除镜像层中的文件时，OverlayFS 在 upperdir 写入一个同名白障项（char device 0/0），合并视图中该文件消失，但镜像层数据原封不动。

与前端已有概念的对比：镜像层 ↔ Git 提交对象（不可变、内容寻址、父链结构；docker pull 相当于 git clone，镜像仓库相当于 npm registry + Git remote 的结合体）。容器进程 ↔ Chrome 多进程架构中的渲染进程：共享同一内核（浏览器内核/OS 内核），各自拥有独立的进程空间与权限边界，区别在于容器隔离由内核直接强制。Dockerfile ↔ package.json + 构建配置：都是声明式描述，但 Dockerfile 声明的是操作系统用户空间、运行时与依赖的最终快照而非依赖关系图。进一步类比『Java 接口 vs TS 接口』：OCI 镜像规范（Open Container Initiative）是运行时契约，类似 Java 接口，任何实现（containerd、runc、CRI-O）都必须遵守；而 Dockerfile 语法是构建期静态声明，类似 TS 接口，构建完成后即被『擦除』，运行时只存在镜像格式本身。

### 3. 基础代码与实战验证
```text
极简验证实验（Node.js 20 镜像，零框架），用于验证分层、写时复制与白障三个核心机制。

Dockerfile（三条指令）：
FROM node:20-alpine
WORKDIR /app
COPY app.js .
CMD ["node", "app.js"]

app.js：
const fs = require('fs');
console.log(fs.readFileSync('/app/data.txt', 'utf8'));

步骤与原理对应：
1. docker build -t layer-demo . —— 构建产生多层：node:20-alpine 基础镜像自身多层 + WORKDIR 层（仅记录工作目录元数据）+ COPY 层（含 app.js 完整内容）。执行 docker history layer-demo 可查看层链、每层大小与创建指令。
2. docker run -it layer-demo sh —— 覆盖 CMD 进入 shell；容器内执行 echo hello > /app/data.txt 后 exit。该写入发生在容器可写层（upperdir），镜像层无任何变化。注意此处不能加 --rm，否则容器退出即被删除，无法执行下一步提交。
3. docker commit $(docker ps -lq) layer-demo-committed —— 将刚才容器（含可写层）整体提交为新镜像。对比 docker history layer-demo-committed 与 docker history layer-demo，会看到多出一层，且该层大小约等于 data.txt 所在文件系统块的实际占用。这直接证明了『容器 = 只读镜像层 + 可写层』。
4. docker run --rm -it layer-demo sh —— 基于原镜像起新容器，内部执行 rm /app/app.js 后 exit。随后执行 docker run --rm layer-demo cat /app/app.js，仍能读到文件内容：删除只是在该容器可写层放置了白障遮蔽项，原镜像层中的 app.js 完好无损。
5. docker inspect layer-demo 查看 RepoDigests 字段：镜像 digest 由全部层内容共同计算得出，任何一层内容变化都会导致 digest 改变，这是镜像不可变性与内容寻址的实证。

补充：docker run -v /host/path:/container/path 使用 bind mount 时，容器对挂载路径的读写直通宿主机，完全绕过 OverlayFS 的 copy-up 机制，这是生产环境持久化数据的标准方式，与镜像分层正交。
```

### 4. 常见误区与进阶思考
误区一：把容器当作轻量虚拟机。容器与宿主机共享内核，不存在内核级隔离，也没有独立的硬件抽象层。容器内看到的 ps 只是被 PID Namespace 过滤后的宿主进程视图；镜像中的用户空间（glibc、/etc）由镜像层提供，但一切系统调用直接进入宿主机内核。由此推出三条铁律：容器内不能运行与宿主机不同内核类型的操作系统；容器内无法修改内核参数（sysctl）与加载内核模块（受宿主约束）；容器的安全边界远低于虚拟机，一个内核漏洞即可从容器逃逸到宿主机。

误区二：认为删除文件可以缩小镜像体积。镜像层是不可变快照，容器内 rm 只是写入白障项，底层数据依旧存在。因此在 Dockerfile 中『先下载大文件，用完再 rm』不会减小镜像，因为中间状态已被持久化为独立层。正确做法是：将下载与删除放在同一条 RUN 指令内（同一层内完成，中间态不落盘），或用多阶段构建只把最终产物 COPY 到新镜像。

进阶思考题：一个 10GB 的镜像，在容器内修改其中 1 个 10MB 的文件后 docker commit，新镜像磁盘增量约为多少？若修改的是 1 个 1GB 的文件呢？请结合 OverlayFS copy-up 的粒度（文件级而非块级）给出答案，并对比虚拟机快照（如 qcow2，块级写时复制）在相同场景下的增量差异——两者的本质区别是什么？
