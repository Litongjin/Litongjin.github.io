---
title: "每日基础技术总结 · 2026-09-15 · Docker 镜像分层与 OverlayFS 的 copy-up 与 merged 视图"
date: 2026-09-15 08:00:00
categories: [技术分享]
tags: ["技术分享", "DevOps 与云原生"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-15 · Docker 镜像分层与 OverlayFS 的 copy-up 与 merged 视图

## 📚 今日主题

> **Docker 镜像分层与 OverlayFS 的 copy-up 与 merged 视图**（DevOps 与云原生）

### 1. 核心概念速览
Docker 镜像分层：镜像由一组只读层(layer)按顺序叠加构成，每层是相对下层的文件系统变更集(新增、修改、删除)。容器启动时，Docker 在镜像层之上追加一个可写层，并由存储驱动(通常 overlay2)将多个 lowerdir 与一个 upperdir 联合挂载为 merged 视图。OverlayFS 是 Linux 内核的联合文件系统(union filesystem)：lowerdir 为多个只读目录，upperdir 为可写目录，workdir 为内核内部使用的空目录，merged 是进程看到的统一目录树。

它解决的问题：在多个容器/镜像间共享只读层，避免复制完整根文件系统；容器写操作按需落在私有可写层，实现快速启动、镜像层复用和存储节省。

核心机制：读时合并(read merge)，写时复制(copy-up)，删除用 whiteout/opaque 标记。目录合并：同名目录的内容合并；同名非目录文件以最高优先级层为准。文件查找：upper 优先，然后 lower 从栈顶到栈底，返回第一个命中。

在计算机/AI 体系中的位置：位于容器运行时与内核文件系统之间，是 OCI 镜像、Docker/containerd、Kubernetes Pod 启动与镜像分发的底层存储抽象。专业工程师必须掌握，因为它直接决定容器启动延迟、镜像拉取量、磁盘占用、写性能、CI 缓存命中，以及调试层泄漏和存储驱动异常。

### 2. 底层原理剖析
路径解析(lookup)伪代码：
function lookup(path):
  if path in upper:
    if upper[path] is whiteout: return ENOENT
    return upper[path]
  for lower in lowers_top_to_bottom:
    if lower[path] is whiteout: return ENOENT
    if lower[path] exists:
      if lower[path] is dir and upper has same dir:
         return merged_dir(upper_entries + lower_entries)
      return lower[path]
  return ENOENT

写打开与 copy-up：
function open_for_write(path):
  if path in upper: return upper[path]
  if path in lower:
    copy_up(path)  # 将 lower 中命中版本完整复制到 upper，保留元数据；不是块级 CoW
    return upper[path]
  return create_in_upper(path)

copy-up 触发条件：open(O_WRONLY/O_RDWR)、truncate、chmod、chown、utimes、setxattr、link 等。目录通常不整体 copy-up，而是在 upper 创建同名目录并合并，必要时使用 opaque 标记隐藏 lower 同目录内容。删除 lower 文件：在 upper 创建 whiteout 特殊文件(字符设备 0/0 或 .wh. 前缀)；删除 lower 目录：在 upper 创建 opaque 目录或 whiteout。重命名 lower 文件：可能 copy-up 后重命名，或 redirect_dir 优化。

OverlayFS 合并规则：
- 非目录：上层覆盖下层，只有最高命中可见。
- 目录：上下层同名目录合并，条目集合并集；若上层目录有 opaque xattr，则隐藏下层同目录全部内容。
- 删除：whiteout 使下层同路径不可见；merged 中表现为不存在。
- 复制：copy-up 后 upper 文件与 lower 文件是两个独立 inode，st_ino/st_dev 可能变化，硬链接需 index=on 维持。

与前端/语言概念对比：
- TS interface 是编译期结构契约，运行时完全擦除；Java interface 是运行时类型契约，通过方法表分发。OverlayFS merged 不是类型契约，而是内核 VFS 路径解析与 inode 合成的运行时结果，取决于 lower/upper 当前状态与挂载参数。
- 前端构建缓存(webpack/vite)按内容哈希缓存产物，不是联合挂载；Docker 层是文件系统树差异，读合并写复制，容器写层与镜像层隔离。
- Git 的 commit/tree/blob 是内容寻址追加模型，层可共享；OverlayFS 是挂载时合成视图，不重写底层对象。

存储驱动路径：/var/lib/docker/overlay2/<layer-id>/diff 为层内容；/var/lib/docker/overlay2/l/<short> 为符号链接；容器目录下 upperdir 为可写层，merged 为挂载点，workdir 为内部工作目录。mount 参数示例：lowerdir=layerN:...:layer1,upperdir=.../diff,workdir=.../work。

性能本质：lookup 需逐层查找，层数越多路径解析成本越高；copy-up 对文件粒度复制，首次写大文件产生延迟与空间放大；readdir 需合并多层目录。metacopy=on 可仅复制元数据，data 仍读 lower，但写数据仍触发完整 data copy-up。

### 3. 基础代码与实战验证
```text
以下命令在 Linux 内核支持 overlay 且具备 root 权限时可直接验证。逐行执行：

# 1. 准备 lower/upper/work/merged 四个目录
mkdir -p /tmp/ovl/{lower,upper,work,merged}

# 2. 构造只读 lower：一个文件和一个目录
echo base > /tmp/ovl/lower/a.txt
mkdir -p /tmp/ovl/lower/dir
echo lower-x > /tmp/ovl/lower/dir/x.txt

# 3. 联合挂载：lowerdir 只读，upperdir 可写，workdir 必须为空且与 upper 同文件系统
mount -t overlay overlay -o lowerdir=/tmp/ovl/lower,upperdir=/tmp/ovl/upper,workdir=/tmp/ovl/work /tmp/ovl/merged

# 4. merged 视图读取 lower 文件：此时未 copy-up，upper 为空
cat /tmp/ovl/merged/a.txt
ls -la /tmp/ovl/upper
stat -c 'inode=%i size=%s file=%n' /tmp/ovl/lower/a.txt /tmp/ovl/merged/a.txt
# merged 中 a.txt 的 inode 来自 lower，因为只读打开不触发 copy-up

# 5. 对 lower 文件执行写打开/追加：触发 copy-up，整个 a.txt 被复制到 upper
# 关键：copy-up 是文件级完整复制，不是只复制被修改的字节
echo modified >> /tmp/ovl/merged/a.txt
cat /tmp/ovl/upper/a.txt
cat /tmp/ovl/lower/a.txt
stat -c 'inode=%i size=%s file=%n' /tmp/ovl/lower/a.txt /tmp/ovl/upper/a.txt /tmp/ovl/merged/a.txt
# merged 中 a.txt 现在指向 upper 的独立 inode，lower 保持原样

# 6. 修改 lower 目录中的文件：同样触发 copy-up，但目录条目在 merged 中合并
echo upper-x >> /tmp/ovl/merged/dir/x.txt
cat /tmp/ovl/upper/dir/x.txt
# upper 中出现 dir/x.txt，lower/dir/x.txt 不变

# 7. 删除 lower 文件：在 upper 创建 whiteout，使 lower 同路径不可见
rm /tmp/ovl/merged/dir/x.txt
ls -la /tmp/ovl/upper/dir
# 可能看到字符设备 0,0 或 .wh.x.txt；这是 whiteout，不是物理删除 lower
cat /tmp/ovl/merged/dir/x.txt  # 返回 No such file or directory

# 8. 在 merged 中新增文件：直接写入 upper，lower 不变
echo upper-y > /tmp/ovl/merged/dir/y.txt
ls -la /tmp/ovl/merged/dir
ls -la /tmp/ovl/upper/dir

# 9. 清理
umount /tmp/ovl/merged
rm -rf /tmp/ovl
```

### 4. 常见误区与进阶思考
误区一：把 copy-up 当成 qcow2/页级 CoW。实际 overlayfs copy-up 是文件粒度完整复制，首次写大文件会复制整个文件到 upper，导致容器可写层磁盘膨胀和写延迟；metacopy 可只复制元数据，但写数据仍复制。删除文件不释放 lower 空间，只在 upper 增加 whiteout，镜像层仍占空间。

误区二：认为 merged 是物理合并目录，或以为容器修改会写回镜像层。实际 merged 是内核动态合成的挂载视图；lower 始终只读，多容器共享 lower，但各自 upper 隔离。另一个常见错误是以为 st_ino 稳定，copy-up 后 inode 改变，硬链接、文件锁、mmap 语义可能受影响。

思考题：若 lowerdir 栈为 lower2:lower1，lower1 和 lower2 都存在路径 p，且 p 是普通文件。现在容器对 /merged/p 执行 open(O_RDWR)。copy-up 会把哪个版本复制到 upper？如果随后在 merged 中删除 p，再重新创建 p，upper 中 whiteout 与新建文件如何交互？请从 lookup、copy-up、whiteout 三阶段解释。答案要点：copy-up 复制 lookup 命中的最高优先级 lower(lower2)版本；删除时 upper 创建 whiteout 遮蔽所有 lower；重新创建时内核会移除 whiteout 并在 upper 创建普通文件，merged 中重新可见新文件。
