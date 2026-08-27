---
title: "每日基础技术总结 · 2026-08-28 · Linux 常用命令与权限管理"
date: 2026-08-28 06:55:48
categories: [技术分享]
tags: ["技术分享", "后端基础（Node.js / Java / Python）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-28 · Linux 常用命令与权限管理

## 📚 今日主题

> **Linux 常用命令与权限管理**（后端基础（Node.js / Java / Python））

### 1. 核心概念速览
Linux 命令是用户态进程通过系统调用请求内核服务的接口；权限管理是内核基于 inode 元数据中的属主(uid)、属组(gid)和 12 位 mode（rwx + suid/sgid/sticky）对进程访问对象进行授权的强制控制机制。它解决多用户、多进程共享同一主机时“谁可以读、写、执行”的隔离问题。在计算机体系中，它位于 OS 的核心，是后端、容器、云原生与 AI 基础设施的基石：任何远程服务器维护、CI/CD、模型服务部署都离不开 shell 命令与权限边界。专业工程师必须掌握，是因为权限模型决定系统的安全边界，而命令用法是感知和操作该模型的唯一途径。

### 2. 底层原理剖析
底层原理：每次文件 I/O 或执行都触发系统调用（sys_open/sys_execve 等），内核 VFS 层通过路径查找定位目标 inode，取出 uid/gid/mode。同时，从当前进程的 task_struct 中获取 fsuid（通常等于 euid）、fsgid 及 supplementary_gids。权限判定采用固定的顺序：若 fsuid 等于 inode->i_uid，则匹配属主类，应用属主权限位；否则若 fsgid 或补充组匹配 inode->i_gid，则应用属组权限位；否则应用 other 权限位。这里不存在“取交集”或“合并权限”，一旦匹配就停止。对于目录，r 允许 list，w 允许创建/删除条目，x 允许 traverse；删除文件实际修改的是父目录的 inode，因此对文件的写权限不决定删除权。进程的 real uid 不参与 DAC 检查，只有 euid/fsuid 和 capabilities 参与。与前端概念对比：TS 的 interface 只是编译期的类型契约，运行时无实体；Linux 权限则是每次系统调用在内核态进行的强制安全检查。类似地，前端的 router 守卫只是用户体验层，真正的鉴权必须在服务端，正如 Linux 不允许用户态进程通过 LD_PRELOAD 或修改内存绕过权限——权限判断是内核唯一可信点。

### 3. 基础代码与实战验证
```text
#!/bin/bash
# 验证 Linux 权限判定：先创建普通文件，并观察其 inode 元数据
touch /tmp/demo
ls -ln /tmp/demo            # -n 显示数字 uid/gid，输出形如 -rw-r--r-- 1 1000 1000 0 ...
chmod 640 /tmp/demo         # 修改 inode->i_mode 为 rw-r-----（八进制 640），只更新元数据不更新数据
sudo -u nobody cat /tmp/demo  # 内核检查：nobody(65534) 不是属主(1000)，也不匹配属组(1000)，
                              # 命中 other 权限位，other 无 r 位，open() 返回 EACCES

# 查看当前进程套用的有效身份和附加组
id
# 输出示例：uid=1000(alice) gid=1000(alice) groups=1000(alice),27(sudo)
# 该信息来自内核为进程维护的 struct credential，每次权限判定只读这里

# 查看 umask：影响新建文件的实际 mode（内核在 creat/mkdir 时执行 mode & ~umask）
umask
# 输出如 0022，则新文件默认为 0644（-rw-r--r--）
```

### 4. 常见误区与进阶思考
误区1：认为 chmod 修改权限是对内容的加密或保护。实际chmod 只修改 inode 中的 mode 字段，不改变数据本身；任何用户若能直接读取磁盘设备或拥有 CAP_SYS_RAWIO，仍可绕过。同理，前端项目中用环境变量隐藏密钥也不是安全机制。
误区2：把权限检查与 Linux 能力机制混淆。root 的 DAC 权限是默认全放行，但现代内核用 capabilities 将 root 权限分解；且 SELinux 的 MAC 会在 DAC 之后再次检查，不能用 chmod 替代。
思考题：若文件的属主与进程的属主相同，但进程的属组也与文件的属组相同，且属主权限比属组权限更严格，内核按哪类权限执行，为什么这样设计？请从 user_ns 和 DAC 检查顺序的角度回答。
