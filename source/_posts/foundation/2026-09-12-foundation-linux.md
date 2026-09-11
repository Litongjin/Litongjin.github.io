---
title: "每日基础技术总结 · 2026-09-12 · Linux 常用命令与权限管理"
date: 2026-09-12 07:02:11
categories: [技术分享]
tags: ["技术分享", "后端基础（Node.js / Java / Python）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-12 · Linux 常用命令与权限管理

## 📚 今日主题

> **Linux 常用命令与权限管理**（后端基础（Node.js / Java / Python））

### 1. 核心概念速览
Linux 是多用户多任务操作系统，其权限管理基于 DAC（自主访问控制）模型：每个文件（包括目录、设备等一切资源）在 inode 中存储属主 UID、属组 GID 和 12 位权限标志（rwx 分别对应属主、属组、其他，加上 setuid/setgid/sticky）。内核在进程每次打开文件时强制执行权限检查，与进程的有效凭证（EUID/EGID）比较，按属主、属组、其他顺序唯一匹配。常用命令（ls、chmod、chown、useradd 等）本质是向 shell 或内核发起系统调用（stat、chmod、chown 等），用于查看和修改这些元数据。解决的问题是：在共享资源环境中实现最小权限隔离，防止越权访问。位置属于操作系统进程管理与文件系统层，是后端工程（部署、容器、CI/CD 脚本）的基石。前端工程师转向后端必须掌握这一点，因为任何服务本质上都是运行在特定用户权限下的进程，文件权限直接影响数据安全与可维护性。

### 2. 底层原理剖析
底层机制：每个进程有真实 UID/GID、有效 UID/GID（EUID/EGID）和附加组列表。访问文件时，内核由 vfs 层获得 inode，取出 i_uid/i_gid 和 i_mode。检查顺序：若进程 EUID 等于文件属主 UID，则读取属主权限位（rwx）；否则若 EGID 或附加组匹配文件属组 GID，则读取组权限位；否则读取其他权限位。权限位不是布尔集合，而是比特掩码：r=4, w=2, x=1。对于目录，r 允许列出条目，w 允许创建/删除条目（需同时有 x），x 允许穿越目录访问内部条目。删除文件的权限取决于父目录的 w+x，而非文件自身权限。chmod 通过系统调用更新 inode 的 i_mode，只有属主或 root 可操作。umask 是进程创建文件时对 mode 的掩码，如 umask 022 使 0666 与 ~022 得到 0644。与前端对比：TypeScript 的接口只在编译期做结构类型检查，不产生任何运行期代码；Linux 权限则是内核在每次 I/O 操作时的强制运行期检查，类似于 HTTP 的 Authorization 中间件，但后者基于网络协议且可绕过，前者不可绕过（除非 root 具有 CAP_DAC_OVERRIDE）。另外，setuid 位允许进程以文件属主身份运行，类似前端代码中通过运行时角色提升，但更危险。

### 3. 基础代码与实战验证
```text
# 创建一个文件，默认权限由当前进程 umask 决定（常见 022，结果为 644，即 rw-r--r--）
touch demo.txt
# 以长格式列出文件元数据：第1个字符为文件类型（- 普通文件），第2-4为属主权限，第5-7为组权限，第8-10为其他权限
ls -l demo.txt
# 使用八进制数字设定精确权限：6=rw-(4+2)，4=r--，0=---。chmod 内部将调用 chmod() 系统调用更新 inode 的 i_mode
chmod 640 demo.txt
# 此时文件权限变为 -rw-r-----（属主可读写，组可读，其他无权限）
# 模拟一个有效 UID 不是文件属主、且不在属组中的进程尝试读取文件：
# 若当前用户有 sudo 权限，执行 sudo -u nobody cat demo.txt，内核在 open() 阶段检查其他权限位，发现为 0，返回 EACCES，shell 输出 Permission denied
sudo -u nobody cat demo.txt || echo 'EACCES: Permission denied'
# 查看当前有效身份，确认进程凭证：id -u 输出 EUID，id -g 输出 EGID
id -u; id -g
```

### 4. 常见误区与进阶思考
常见误区1：认为文件的所有权或权限决定能否删除文件。实际上删除文件的操作对象是父目录，需要父目录的写+执行权限，文件自身权限只影响文件内容修改。因此一个只读文件如果其父目录可写，可以被 rm 删除。
常见误区2：混淆有效 UID 与真实 UID。在普通 shell 中两者相同，但通过 setuid 程序运行时，进程的 EUID 变为程序属主 UID，而 RUID 仍是调用者；内核检查文件权限时使用 EUID。例如 /usr/bin/passwd 有 setuid 位，普通用户可写 /etc/shadow。若不理解这一点，会错误地认为所有进程都以启动用户身份访问文件。
思考题：如果一个目录的权限为 777 且有 sticky 位（如 /tmp），为什么普通用户只能删除自己属主的文件，而不能删除其他用户的文件？请分析内核在 unlink 时对目录 sticky 位的检查逻辑，以及它如何用 EUID 和文件属主 UID 做判断。
