---
title: "每日基础技术总结 · 2026-09-09 · Docker 镜像分层与 OverlayFS 的 copy-up 与 merged 视图"
date: 2026-09-09 08:00:00
categories: [技术分享]
tags: ["技术分享", "DevOps 与云原生"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-09 · Docker 镜像分层与 OverlayFS 的 copy-up 与 merged 视图

## 📚 今日主题

> **Docker 镜像分层与 OverlayFS 的 copy-up 与 merged 视图**（DevOps 与云原生）

### 1. 核心概念速览
镜像分层：Docker 镜像由一组只读层（layer）按顺序堆叠生成，每一层本质是上一层文件系统状态与当前状态的差量（diff），通常以 tar 归档存储。OverlayFS 是一种 Linux 联合文件系统，核心机制是将多个目录（lowerdir 与 upperdir）合并投影为统一的 merged 视图：lowerdir 是只读的镜像层链，upperdir 是可写的容器层，读操作按层从高到低查找，写操作要么直接落在 upperdir，要么先执行 copy-up。copy-up 是 OverlayFS 的写时复制机制：当对 lowerdir 中已存在的文件进行修改或删除时，内核先把整个文件（含内容与元数据）复制到 upperdir，再在 upperdir 上执行变更，此后 upperdir 的副本遮蔽 lowerdir 的原文件。该机制解决的核心问题是：多个容器共享镜像层时可只读复用、隔离副本，仅在真实写入时产生增量，从而节省存储与网络带宽。它位于容器运行时存储栈底层，是 Docker overlay2 storage driver 的基石，直接决定镜像构建、容器启动与运行时 I/O 的性能。专业工程师必须掌握它，因为层的设计与文件变更的位置会直接放大镜像体积、引发缓存失效、造成写放大；不理解 copy-up 就难以诊断容器磁盘占用与 I/O 热点。整个云原生体系中的镜像分发、registry 层复用、根文件系统只读安全模型都建立在这一机制之上。

### 2. 底层原理剖析
OverlayFS 由三种视图构成：lowerdir（若干只读目录）、upperdir（唯一可写目录）、merged（挂载后呈现的合并视图）。Docker overlay2 driver 把每一层镜像目录按从底到顶的顺序拼接为 lowerdir，容器顶层可写目录作为 upperdir。挂载后，merged 视图对进程呈现一个普通目录树。路径解析是自顶向下逐层查找：先查 upperdir，再依次查由新到旧的 lowerdir，第一个命中路径生效；因此上层的同名文件/目录遮蔽下层。当进程对 merged 视图中的某文件发起 write、truncate、rename、unlink 等修改类操作，且该文件实际存在于 lowerdir 时，OverlayFS 触发 copy-up：以文件为粒度，把文件内容及时间戳、权限、xattr 等元数据完整复制到 upperdir，随后在 upperdir 上执行原操作。此后该文件在 upperdir 中有了实体，后续读写不再访问 lowerdir，而 lowerdir 原文件保持不变直到所属镜像层被删除。删除操作有特殊语义：若删除的是 lowerdir 中的文件，OverlayFS 并不修改 lowerdir，而是在 upperdir 创建一个 whiteout（字符设备，设备号 0/0），merged 视图对该路径呈现为不存在；若删除的是目录且 lowerdir 存在同名目录，则会创建 opaque 标记。因此容器层最终包含：本层新建文件、被修改文件的副本、以及删除标记 whiteout——这正是“层是 diff”的落地形式。从数据结构看，这是一条带遮蔽规则的单链表加上写时复制，查找复杂度 O(层数)，且 copy-up 是整文件复制，代价与写入字节数无关，只与文件大小相关。

与前端知识对比：这与 JavaScript 原型链高度同构。lowerdir 等价于原型链上各级 prototype 对象，upperdir 等价于实例的自有属性，merged 视图等价于运行时属性解析结果。读属性沿 [[Prototype]] 链向上查找，命中即返回，等同于合并视图的层级遮蔽；在实例上给一个来自原型的属性赋值，不会修改 prototype，而是在实例上创建同名自有属性并遮蔽原型，这正是 copy-up 的语义；delete 一个来自原型的属性不会影响原型，需要额外手段（如不可配置属性或 Proxy 拦截）才能阻止原型属性重新暴露，对应 OverlayFS 必须用 whiteout 来遮蔽 lowerdir 文件。这里与“Java 的 interface 与 TypeScript 的 interface 同名但语义域不同”类似：Docker 镜像层与 OverlayFS 层也是两个层次的概念——镜像层是存储分发与缓存复用单元（内容寻址的 tar diff），OverlayFS 层是内核运行时目录合并与写时复制单元；弄清这一层抽象归置，才不会在排查问题时把镜像构建期与容器运行期机制混为一谈。

### 3. 基础代码与实战验证
```text
以下为纯 shell 验证脚本，需要 root 权限与内核 overlay 支持。核心观察点：读不触发 copy-up，写触发整文件复制，删除产生 whiteout。

# 1. 准备目录结构
mkdir -p /tmp/lower /tmp/upper /tmp/work /tmp/merged
echo 'lower: original' > /tmp/lower/file.txt

# 2. 挂载 OverlayFS：lowerdir 提供只读基础视图，upperdir 捕获所有写入
mount -t overlay overlay \
  -o lowerdir=/tmp/lower,upperdir=/tmp/upper,workdir=/tmp/work \
  /tmp/merged

# 3. 读操作：只走查找路径，不触发 copy-up，upper 目录保持为空
cat /tmp/merged/file.txt          # 输出 lower: original
ls -A /tmp/upper                  # 无输出，说明没有发生复制

# 4. 写操作：对 merged 视图中来自 lowerdir 的文件写入，内核触发整文件 copy-up
#    实际过程：先完整复制 lowerdir/file.txt 到 upperdir/file.txt，再在 upperdir 上执行写入
echo 'upper: modified' > /tmp/merged/file.txt

# 5. 验证 copy-up 结果：lower 原文件未被改动，upper 出现整个文件副本
cat /tmp/lower/file.txt           # lower: original
cat /tmp/upper/file.txt           # upper: modified  —— 副本已遮蔽 lower

# 6. 删除操作：并不删除 lowerdir 中的文件，而是在 upperdir 留下 whiteout 遮蔽条目
rm /tmp/merged/file.txt
ls -la /tmp/upper/file.txt        # 不存在（upper 副本已被删除）
ls -la /tmp/lower/file.txt        # lower 原文件仍然存在
cat /tmp/merged/file.txt          # No such file or directory
ls -la /tmp/upper/file.txt 的等价条目应显示为：
# crw------- 1 root root 0, 0 ... file.txt   （whiteout 字符设备）

# 7. 卸载清理
umount /tmp/merged

这段脚本直观验证了四件事：(1) merged 视图只是投影，不是实体目录；(2) copy-up 是文件级的，修改一个字节也要复制整个文件；(3) lowerdir 在容器生命周期内完全只读；(4) 删除操作以 whiteout 方式实现，实际数据仍留在低层。
```

### 4. 常见误区与进阶思考
误区 1：认为容器内修改文件是“就地写入镜像层”。实际上任意写操作都会触发整文件 copy-up，文件有多大就复制多少；Dockerfile 中执行一条 `RUN echo xxx >> large.bin` 会把整个大文件复制进新层，镜像膨胀是必然。同理，在 Dockerfile 内先创建 10GB 文件再删除，删除产生的 whiteout 只负责遮蔽可见性，10GB 数据仍保留在历史层中，镜像体积不会变小；这正是必须重视 .dockerignore 与多阶段构建的深层原因。

误区 2：把 merged 视图当作普通文件系统，忽略查找顺序与遮蔽规则。文件与目录行为不同：目录会跨层合并内容，文件则是最高层命中即屏蔽低层存在；层数越多，路径查找越慢。copy-up 还会递归地把父目录链完整复制到 upperdir，若上层目录结构复杂，会进一步放大 I/O。若忽略这些差异，很容易在设计镜像层时让缓存频繁失效，或在运行期产生不可预期的读放大与写放大。

进阶思考题：假设镜像 lowerdir 中存在一个 10GB 的大文件 data.bin，容器内依次执行 `echo hello >> data.bin` 与 `rm data.bin`。请结合 copy-up 与 whiteout 推演全过程：追加写入是否触发整文件复制？删除动作回收的是哪一份副本？镜像层中那 10GB 是否仍然占用磁盘？最终容器可写层（upperdir）的大小近似是多少？如果 1000 个容器并发执行同样操作，宿主机磁盘峰值占用与最终占用如何变化？真正的理解者应当能给出“写入期每容器产生 10GB 写放大、删除后 upper 副本立即释放、lower 层 10GB 持续存在、最终每容器 upperdir 仅剩一个不足 1KB 的 whiteout”的完整链路。
