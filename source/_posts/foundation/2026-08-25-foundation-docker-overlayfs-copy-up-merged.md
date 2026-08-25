---
title: "每日基础技术总结 · 2026-08-25 · Docker 镜像分层与 OverlayFS 的 copy-up 与 merged 视图"
date: 2026-08-25 08:00:00
categories: [技术分享]
tags: ["技术分享", "DevOps 与云原生"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-25 · Docker 镜像分层与 OverlayFS 的 copy-up 与 merged 视图

## 📚 今日主题

> **Docker 镜像分层与 OverlayFS 的 copy-up 与 merged 视图**（DevOps 与云原生）

### 1. 核心概念速览
Docker 镜像由多个只读层（layer）叠加构成，每一层代表一组文件系统操作（diff）。OverlayFS 是一种联合文件系统（union filesystem），将多个 lowerdir 与一个 upperdir 合并挂载为 merged 视图。copy-up（写时复制）是 OverlayFS 的核心机制：当进程修改 merged 视图中位于 lowerdir 的文件时，OverlayFS 先将该文件完整复制到 upperdir，再在副本上执行修改，从而保证 lowerdir 的只读性和镜像层共享性。该机制解决三个核心问题：镜像层复用与存储空间最小化、容器间隔离、以及容器可写层与只读镜像层的统一视图。在计算机体系结构中，它位于操作系统存储栈（VFS -> 具体文件系统）与容器运行时（runc/containerd）之间，是理解镜像拉取、构建缓存、容器启动性能与磁盘 IO 行为的基石。专业工程师必须掌握，因为任何容器性能问题、镜像瘦身策略、层缓存失效的本质都可以追溯到这一分层与 copy-up 机制。

### 2. 底层原理剖析
OverlayFS 挂载参数包含 lowerdir、upperdir、workdir 和 merged。lowerdir 可以包含多个目录（以冒号分隔），顺序为从底层到顶层；merged 是动态合成的视图，不占用额外存储。文件访问遵循 topmost-first 规则：在 merged 中查找文件时，从最高层 lowerdir 开始向下，再到 upperdir；实际读取时，若 upperdir 存在同名文件，则优先使用 upperdir。写操作触发 copy-up：对 merged 中某个文件进行写打开（如 open(O_WRONLY)）时，VFS 调用 OverlayFS 的 file_operations -> write_iter，OverlayFS 首先检查该文件是否已在 upperdir，若不在，则执行 ovl_copy_up：分配 upperdir 中相同路径的 inode，将 lower 文件的内容和元数据（包括权限、时间戳、xattr）复制到 upperdir，然后重定向当前文件句柄到新副本。此后写操作直接在 upperdir 副本上进行。目录的 copy-up 是递归的，且只复制路径上缺失的目录，而非整个目录树。删除操作通过 whiteout 实现：在 upperdir 创建同名的字符设备（0/0）或带 'c' 标记的 overprivileged 文件，使 merged 视图中该文件呈现为已删除，而 lowerdir 原始文件保持不变。对比前端已有概念：Java 接口与 TypeScript 接口的本质区别在于，Java 接口在运行时仍是类型系统的一部分（如 instanceof 检查），而 TS 接口是纯编译期结构类型，运行时被完全擦除。这与 OverlayFS 的 merged 视图有相似之处：merged 并非真实存在的实体，而是由 lowerdir 和 upperdir 动态合成；它类似于 TS 的结构类型——只要目录结构兼容（同名文件存在），即可呈现为统一视图，而无需显式声明合并关系。另一方面，镜像层之间的继承关系又像 Java 接口的显式实现：每一层在 Dockerfile 中显式继承自上一层，但 OverlayFS 的 lowerdir 不要求层间有指针关系，而是通过内容寻址的存储（如 sha256）来关联。这种'运行时合成'与'编译期擦除'的对比，帮助前端工程师理解联合文件系统的动态性与容器的不可变基础设施之间的张力。

### 3. 基础代码与实战验证
```text
以下为手动模拟 OverlayFS 挂载与 copy-up 的极简实验（Linux 环境，需 root 权限）：

# 1. 创建实验目录
mkdir -p /tmp/overlay/{lower,upper,work,merged}

# 2. 在 lower 层创建初始文件
echo 'base layer' > /tmp/overlay/lower/file.txt

# 3. 挂载 OverlayFS
# lowerdir 是只读层；upperdir 是容器可写层；workdir 必须与 upperdir 同文件系统，用于原子操作
mount -t overlay overlay \
  -o lowerdir=/tmp/overlay/lower,upperdir=/tmp/overlay/upper,workdir=/tmp/overlay/work \
  /tmp/overlay/merged

# 4. 读取 merged 视图（此时文件来自 lower 层）
cat /tmp/overlay/merged/file.txt  # 输出 'base layer'

# 5. 写修改触发 copy-up：修改 merged 中的文件
# 底层过程：VFS 收到写请求，OverlayFS 发现 upperdir 中不存在 file.txt，
# 于是将 lower 中的 file.txt 完整复制到 upper，然后修改副本
echo 'modified' > /tmp/overlay/merged/file.txt

# 6. 验证 copy-up 结果
cat /tmp/overlay/upper/file.txt    # 输出 'modified'（副本已修改）
cat /tmp/overlay/lower/file.txt    # 输出 'base layer'（原始层未动）

# 7. 删除操作产生 whiteout
rm /tmp/overlay/merged/file.txt
# 此时 upper 目录中会出现一个名为 file.txt 的字符设备（可用 ls -l 查看）
# 这个 whiteout 掩盖了 lower 中的同名文件，但 lower 文件实际仍存在
ls -l /tmp/overlay/upper/file.txt   # 显示 c--------- ... file.txt

# 8. 卸载
umount /tmp/overlay/merged
```

### 4. 常见误区与进阶思考
误区 1：误以为容器修改文件会修改镜像层。实际上 copy-up 保证任何写操作都发生在 upperdir（容器可写层），镜像层始终保持只读，这保障了镜像的不可变性和多个容器共享同一镜像时的安全。误区 2：误以为 copy-up 只复制被修改的块或使用写时共享（如 reflink）。OverlayFS 的 copy-up 是文件级别的完整复制（可能后续版本有优化），对于大文件修改哪怕只写一个字节，也会将整个文件复制到 upper 层，导致磁盘空间和 IO 开销成倍增加。这也是为何容器内频繁写大文件会造成明显的性能下降。

进阶思考题：假设一个容器进程对位于 lower 层的一个大文件执行 mmap(MAP_SHARED) 映射后，再通过映射内存写入数据。请问 OverlayFS 如何触发 copy-up？这个过程中的页错误（page fault）处理与普通文件写入有何不同？请从 VFS 与具体文件系统的交互层面解释，这能帮助你真正理解 copy-up 的触发边界。
