---
title: "每日基础技术总结 · 2026-07-03 · Docker 镜像分层与 OverlayFS 的 copy-up 与 merged 视图"
date: 2026-07-03 08:00:00
categories: [技术分享]
tags: ["技术分享", "DevOps 与云原生"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-07-03 · Docker 镜像分层与 OverlayFS 的 copy-up 与 merged 视图

## 📚 今日主题

> **Docker 镜像分层与 OverlayFS 的 copy-up 与 merged 视图**（DevOps 与云原生）

### 1. 核心概念速览
Docker 镜像分层是指镜像由一系列只读的 diff 层（layer）按父子关系堆叠构成，每一层记录相对下一层的文件系统变更（新增、修改、删除）。容器运行时，存储驱动在只读层之上叠加一个可写层（container layer），形成容器进程可见的完整根文件系统。OverlayFS 是当前主流的联合文件系统存储驱动（overlay2），本质是在 VFS 层之下将多个目录（lowerdir 只读层 + upperdir 可写层 + workdir 元数据工作区）联合成一个 merged 挂载视图。copy-up 是 OverlayFS 实现写时复制（CoW）的核心机制：当对仅存在于 lowerdir 中的文件发起写操作时，VFS 截获该操作，先将整个文件连同元数据复制到 upperdir，再在副本上执行写。该机制保证镜像层绝对不可变、多容器共享镜像底层互不污染、容器写入不回传镜像层。在计算机体系中的位置：属于操作系统存储栈中文件系统层与块设备层之间的联合文件系统抽象；在云原生体系中，它是容器运行时与镜像分发之间的文件系统基础，直接影响镜像构建效率、容器启动速度和运行时读写性能。专业工程师必须掌握，因为优化镜像体积（层合并与缓存命中）、诊断容器写性能瓶颈（copy-up 开销）、理解 Docker commit 的 diff 语义、排查 whiteout 引发的文件"假删除"问题，全部建立在对该机制的精确认知之上。

### 2. 底层原理剖析
OverlayFS 挂载模型：

mount -t overlay overlay -o lowerdir=/A:/B,upperdir=/upper,workdir=/work /merged

- lowerdir 可含多个目录，冒号分隔，优先级从左到右递减（A 中文件优先于 B）
- upperdir 为可写层，所有写入发生在该层
- workdir 与 upperdir 必须在同一文件系统，用于原子操作与 copy-up 的临时元数据
- merged 是联合视图，进程通过它访问文件

Docker overlay2 驱动映射：
- 镜像层链：每层镜像解包为独立目录，按从底到顶顺序拼接为 lowerdir（底部层在右）
- 容器可写层 = upperdir
- 容器挂载点 = merged

文件查找（lookup）流程：
1. 先查 upperdir，命中即返回（遮蔽所有 lowerdir 同名文件）
2. 未命中则按 lowerdir 优先级从高到低逐个查找
3. 全部未命中返回 ENOENT

写操作触发 copy-up 的伪代码：

function write(path, data):
    if !exists_in_upperdir(path):
        src = lookup_in_lowerdir(path)
        if src exists:
            copy_up(src, upperdir/path)
            // 复制内容、权限、uid/gid、时间戳、xattr
            // 复制是文件级整文件拷贝，不是块级
    write_to(upperdir/path, data)

copy-up 之后：
- lowerdir 原文件保持不动
- upperdir 出现完整副本，后续所有读写命中该副本
- 对 1GB 文件写 1 字节，物理上先复制 1GB 再修改——这是 OverlayFS 与块级 CoW（Device Mapper、ZFS）的本质差异

删除操作与 whiteout：
- 删除 merged 中的文件：先检查该文件在哪个层命中
- 若命中在 lowerdir（只读），则无法直接删除，改为在 upperdir 创建 whiteout 字符设备（mknod c 0 0）
- merged 视图遇到 whiteout 时屏蔽 lowerdir 对应文件，表现为"已删除"
- 若之后 whiteout 被删除，lowerdir 文件在 merged 中重新可见（"复活"）
- 若文件本就在 upperdir，则直接 unlink，不产生 whiteout

Docker 语义映射：
- docker commit 将 upperdir 内容打包为新的只读层
- docker build 每条 RUN/COPY 指令产生一个新层，层内包含该指令对文件系统的 diff
- 多容器共享镜像：所有容器共享同一组 lowerdir，各自拥有独立 upperdir，互不可见

与前端知识体系的异同：
- 查找遮蔽语义 = JavaScript 原型链：upperdir 等价于实例自身属性，lowerdir 等价于原型链，查找从自身到原型深处，命中即遮蔽。给原型属性赋值不会改变原型，而是在实例上创建新属性——这正是 copy-up 的语义。区别在于原型链只有一条链，OverlayFS 的 lowerdir 是多层栈。
- 镜像层不可变 = Git commit 对象不可变、内容寻址；容器层 = 工作区；copy-up 近似于 fork() 的 CoW 页面语义，但粒度不同（整个文件 vs 内存页）。
- whiteout = 逻辑墓碑而非物理删除，类似浏览器 HTTP 缓存中显式清除后仍需墓碑标记才能遮蔽下层缓存；也类似 TypeScript 中无法真正删除基类成员、只能在派生类中遮蔽。
- 与 Java 接口和 TS 接口的差异类似：底层机制（VFS 联合挂载）是确定性的、有状态的、依赖于内核实现，而前端类型接口是编译期无运行时的结构性契约。OverlayFS 的行为是可观测的运行时事实，接口则只是编译期约束。

### 3. 基础代码与实战验证
```text
以下命令序列在 root 权限下直接验证 OverlayFS 分层与 copy-up/whiteout 机制：

# 1. 构建实验目录
mkdir -p /tmp/ovl/{lower,upper,work,merged}
echo "original" > /tmp/ovl/lower/file.txt
echo "lower-only" > /tmp/ovl/lower/keep.txt

# 2. 挂载 OverlayFS
mount -t overlay overlay \
  -o lowerdir=/tmp/ovl/lower,upperdir=/tmp/ovl/upper,workdir=/tmp/ovl/work \
  /tmp/ovl/merged

# 3. 初始状态：merged 可见 lower 全部文件
cat /tmp/ovl/merged/file.txt   # original

# 4. 对 lower 中的文件执行写（触发 copy-up）
echo "modified" > /tmp/ovl/merged/file.txt

# 5. 验证 copy-up：upper 出现副本，lower 原文件不变
cat /tmp/ovl/upper/file.txt   # modified
cat /tmp/ovl/lower/file.txt   # original

# 6. 删除 lower 中的文件（触发 whiteout）
rm /tmp/ovl/merged/keep.txt

# 7. 验证 whiteout：upper 中出现字符设备，merged 中不可见
ls -l /tmp/ovl/upper/keep.txt   # c 0 0 设备
ls /tmp/ovl/merged/keep.txt     # 不存在

# 8. 清理
umount /tmp/ovl/merged

Docker 层面验证：

docker pull nginx:alpine
docker history nginx:alpine   # 查看镜像层链及大小

docker run -d --name ovl-demo nginx:alpine sleep 3600
docker inspect ovl-demo --format '{{.GraphDriver.Data.LowerDir}}'
docker inspect ovl-demo --format '{{.GraphDriver.Data.UpperDir}}'
docker inspect ovl-demo --format '{{.GraphDriver.Data.MergedDir}}'

docker exec ovl-demo touch /newfile
# 查看 UpperDir 中出现了 /newfile
docker exec ovl-demo rm /etc/nginx/nginx.conf
# 查看 UpperDir 中出现对应 whiteout 条目

关键观察点：
- copy-up 的粒度是"文件"，不是"数据块"
- whiteout 在 upperdir 中以字符设备呈现，而非普通删除
- merged 视图是"逻辑拼接"，不是"物理合并"
```

### 4. 常见误区与进阶思考
误区 1：把 copy-up 当作块级写时复制。
OverlayFS 的 copy-up 是文件级整文件复制。容器内对一个 1GB 文件执行单字节修改，物理上先将整个文件复制到 upperdir（1GB 读 + 1GB 写），然后才修改。这导致严重 IO 放大。块级 CoW（Device Mapper、Btrfs、ZFS）开销小得多。生产环境中，对高频写的大文件（数据库文件、日志），应使用 volume 挂载绕开 overlay，或将大文件置于镜像构建早期层，避免落入 lowerdir 后在容器层触发 copy-up。

误区 2：认为删除 lowerdir 中的文件会释放镜像空间 / 减小镜像体积。
删除 merged 视图中的 lowerdir 文件，仅在 upperdir 创建 whiteout 墓碑。底层数据块仍完整存在于镜像层中，镜像体积不会减小。这正是 docker build 中先下载大文件再 rm 的层中删除不会减小镜像体积的根本原因。若真正缩小镜像，必须在更早的层中避免写入，或重建镜像链。

思考题（检验是否真正理解底层逻辑）：

OverlayFS 挂载场景：
- lowerdir 中存在文件 A，内容 "v1"
- 对 merged 中的 A 执行 echo "v2" > A（触发 copy-up，upperdir 出现 A 副本 v2）
- 对 merged 中的 A 执行 rm（触发 whiteout，upperdir 出现 A 的 whiteout）

问：
1. 此刻 upperdir 中与 A 相关的对象有哪些？lowerdir 中呢？
2. merged 视图中 A 是否可见？
3. 删除 upperdir 中的 whiteout 后，merged 视图中 A 以什么内容出现？
4. 删除 upperdir 中的 A 副本（保留 whiteout）后，merged 视图中 A 是否可见？
5. 这一机制如何解释 docker build 中"删除大文件但镜像体积未减小"的现象？

答案线索：
1. upperdir：A 的完整副本（v2）+ A 的 whiteout（字符设备 c 0 0）；lowerdir：A（v1）原封不动。
2. 不可见。whiteout 在 lookup 中优先级最高，直接返回 ENOENT，遮蔽 upperdir 副本与 lowerdir 原文件。
3. 以 "v2" 出现。whiteout 移除后，lookup 先命中 upperdir 的 A 副本，遮蔽 lowerdir 的 v1。
4. 可见，以 "v1" 出现。upperdir 无 A 副本也无 whiteout，lookup 回落到 lowerdir。
5. 因为 rm 在 upperdir 创建的是 whiteout，不是物理删除 lowerdir 数据。镜像层只读，删除只能以墓碑形式记录在上层，数据块仍存在，镜像体积不减小，docker history 仍显示包含该大文件的层的大小。
