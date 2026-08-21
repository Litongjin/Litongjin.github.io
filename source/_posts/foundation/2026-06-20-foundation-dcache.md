---
title: "每日基础技术总结 · 2026-06-20 · 文件系统目录项缓存与 dcache 结构"
date: 2026-06-20 08:00:00
categories: [技术分享]
tags: ["技术分享", "操作系统基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-06-20 · 文件系统目录项缓存与 dcache 结构

## 📚 今日主题

> **文件系统目录项缓存与 dcache 结构**（操作系统基础）

### 1. 核心概念速览
目录项缓存（dcache）是操作系统 VFS 层维护的、将路径分量（dentry）映射到 inode 的内存索引结构，其本质是一个以父目录 dentry + 文件名元组为键、以 dentry 对象为值的哈希表，并附带 LRU 回收与引用计数机制。它解决的核心问题是：路径解析（path lookup）时避免每次逐级发起磁盘 I/O 读取目录数据，而是通过缓存每个路径分量对应的 dentry，将路径解析从 O(深度×磁盘IO) 降为 O(深度×内存哈希查找)。dcache 位于 VFS 与具体文件系统之间，是 Linux 内核中所有文件访问的必经之路。专业工程师必须掌握它，因为它是理解文件系统性能特征、缓存一致性（cache coherency）、目录操作原子性以及分布式文件系统客户端缓存设计的基石；同时，它与前端工程中的状态缓存（如 Redux 的 selector 缓存、HTTP 缓存）有本质区别——dcache 是内核态、全局共享、受硬性一致性约束的缓存，而前端缓存多为用户态、可自由失效的优化。

### 2. 底层原理剖析
dcache 的核心数据结构是 struct dentry。每个 dentry 代表路径中的一个分量，比如 /a/b/c 会产生三个 dentry：根目录 '/'、'a'、'b'，最终 'c' 可以是 dentry 或挂在 inode 上。关键字段：d_parent（父 dentry 指针）、d_name（qstr，包含名字和哈希）、d_inode（指向 inode，若为空则说明该 dentry 是负缓存）、d_hash（哈希链表节点）、d_lru（回收链表）、d_refcount（引用计数）、d_subdirs（子目录链表，用于遍历）。

路径解析过程（以 open('/a/b/c') 为例）：
1. 从当前进程的 fs->root 或 cwd 获取起始 dentry（已被持有）。
2. 对路径第一个分量 'a'，在起始 dentry 的 d_subdirs 中或通过全局哈希表（dentry_hashtable，以父 dentry 地址和 name hash 为键）查找。若命中，增加 refcount 并继续；若未命中，则调用实际文件系统的 lookup 操作，从磁盘读取目录项，构造新 dentry 插入哈希表和子目录链表，同时与 inode 关联。
3. 逐级向下，直到路径结束。最终 dentry 对应的 inode 会被打开。

机制要点：
- 哈希表解决快速查找，但同一父目录下的子目录链表用于顺序遍历和回收。
- 每个 dentry 生命周期由 refcount 管理：路径解析时持有，文件打开后由 file 结构持有；释放时先脱离哈希表，再回收到 LRU。
- 负缓存（negative dentry）：当文件不存在时，内核仍会缓存一个 dentry 但 d_inode 为 NULL，避免重复的磁盘 ENOENT 查询，但必须维护目录变更时的失效（invalidate）。
- 目录变更（create/unlink/rename）时，VFS 调用 d_add、d_drop、d_delete 等接口更新 dcache。如果 dcache 与实际文件系统不一致，会导致数据不一致，因此所有文件系统必须通过 VFS 的 dentry 操作接口来修改目录项，而不是直接修改磁盘结构。

与前端概念对比：前端开发中的依赖缓存（如 webpack 的 persistent cache）或 Redux 的 memoized selector 是基于“纯函数输入输出”的缓存，失效条件是输入变化；而 dcache 是系统全局共享的可变状态，失效条件是外部事件（目录修改）。类似地，前端 React 的 key 可以类比 dentry 的 hash，但 React 的 reconciler 是单线程、可随时重建的，而内核 dcache 必须考虑并发、内存压力回收和硬一致性。

### 3. 基础代码与实战验证
```text
// 以下为 Linux 内核 dentry 结构的精简伪代码（非真实语法），用于说明底层字段与操作
struct dentry {
    struct dentry *d_parent;       // 父目录 dentry，路径解析时向上回溯的关键
    struct qstr d_name;            // 分量名，内含 hash 值，避免每次计算字符串哈希
    struct inode *d_inode;         // 关联的 inode；若为 NULL，表示负缓存
    struct hlist_node d_hash;      // 全局哈希表节点，键为 (d_parent, d_name.hash)
    struct list_head d_lru;        // 未被引用时的 LRU 节点，内存紧张时回收
    atomic_t d_refcount;           // 引用计数：路径解析持有、file 持有、哈希表持有
    struct list_head d_subdirs;    // 子目录链表，供 readdir 和遍历使用
};

// 路径解析的极简伪代码：
struct dentry *path_lookup(struct dentry *parent, const char *name) {
    // 1. 计算 name 的哈希
    u32 hash = full_name_hash(name);
    // 2. 在哈希表中查找 (parent, hash) 对应的 dentry
    struct dentry *d = hash_lookup(parent, hash, name);
    if (d) {
        // 3. 命中：增加引用计数，防止被回收
        atomic_inc(&d->d_refcount);
        // 4. 如果 d->d_inode 为 NULL，且负缓存未被禁用，直接返回
        return d;
    }
    // 5. 未命中：调用具体文件系统的 inode_operations->lookup
    struct inode *inode = parent->d_inode->i_op->lookup(parent->d_inode, name);
    if (inode) {
        // 6. 创建新 dentry 并与 inode 绑定，插入哈希表和子目录链表
        d = d_alloc(parent, name);
        d_add(d, inode);  // 设置 d_inode，并加入哈希表
    } else {
        // 7. 文件不存在：创建负缓存 dentry，设置 d_inode = NULL
        d = d_alloc(parent, name);
        d_add_negative(d);  // 同样插入哈希表，但不关联 inode
    }
    atomic_inc(&d->d_refcount);
    return d;
}

// 上述代码验证了 dcache 的两个核心行为：
// - 所有路径解析先查缓存，缓存未命中才触发磁盘/文件系统操作；
// - 负缓存同样占用内存，因此删除文件后需要 d_drop() 使对应 dentry 失效。

// 若在用户态验证，可对比：频繁 stat() 一个文件时，第一次耗时含磁盘 I/O，后续耗时显著下降；
// 而 strace 中观察到的 getdents/readdir 次数减少，间接证明 dcache 生效。
```

### 4. 常见误区与进阶思考
误区一：认为 dcache 缓存的是文件内容。实际上 dcache 只缓存路径分量到 inode 的映射，不缓存文件数据。文件内容由 page cache（页缓存）管理。初学者混淆两者，导致在调试文件读写性能时错误地分析 dcache。前端类比：把 HTTP 响应体缓存与 URL 路由缓存混为一谈。
误区二：认为只要 dentry 存在，文件就一定存在。负缓存（negative dentry）的存在意味着 dentry 可以指向一个不存在的文件。如果程序在缓存未失效时依赖 dentry 判断文件存在性，会得到错误结果。例如在用户态，通过 open() 返回 ENOENT 后，立即再次 open 可能由于负缓存导致仍然 ENOENT，但此时文件已被其他进程创建；内核通过目录变更事件（inotify）或 d_delete 来使负缓存失效。

进阶思考题：在 NFS（网络文件系统）中，客户端 dcache 需要与服务器保持一致性。如果客户端修改了一个文件的权限位（chmod），该文件的 dentry 是否必须失效？为什么？请结合 dcache 的字段（d_inode 指向的 inode 的属性缓存）和 VFS 的 revalidation 机制（例如 dentry->d_op->d_revalidate）分析：dcache 的缓存粒度是否只包含路径结构？inode 的属性缓存是否属于 dcache 的一部分？如果不失效，服务器端修改了文件数据，客户端的 mmap 映射会发生什么？这个问题要求你从 VFS 层与具体文件系统层的协作关系来理解 dcache 的真正边界。
