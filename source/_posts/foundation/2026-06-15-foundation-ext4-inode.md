---
title: "每日基础技术总结 · 2026-06-15 · ext4 的 inode 与块寻址：直接/间接块"
date: 2026-06-15 08:00:00
categories: [技术分享]
tags: ["技术分享", "操作系统基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-06-15 · ext4 的 inode 与块寻址：直接/间接块

## 📚 今日主题

> **ext4 的 inode 与块寻址：直接/间接块**（操作系统基础）

### 1. 核心概念速览
inode（索引节点）是 ext4 文件系统中描述文件元数据与数据块位置的核心结构，本质是一个定长记录（ext4 中默认为 256 字节），存储文件模式、属主、时间戳、大小、块计数以及块寻址数组。块寻址解决的是『给定文件偏移，如何定位到物理磁盘块』的问题。ext4 的经典寻址方式采用 12 个直接块指针 + 1 个一级间接块指针 + 1 个二级间接块指针 + 1 个三级间接块指针的层级结构（现代 ext4 也支持 extent 树，但直接/间接块仍是理解文件系统演进的基础）。该机制位于操作系统存储栈中虚拟文件系统（VFS）之下、块设备层之上，是文件系统布局与磁盘 I/O 的枢纽。专业工程师必须掌握它，因为任何涉及磁盘容量规划、碎片化分析、大文件读写性能、文件系统调试（如 debugfs）以及存储引擎设计（数据库、日志系统）的工作，最终都要回到 inode 与块寻址的物理现实上；同时，它也是理解后续 ext4 extent 映射、btrfs/ZFS 等现代写时复制文件系统的必要前提。

### 2. 底层原理剖析
ext4 将磁盘划分为块组（block group），每个块组包含 inode 表与数据块区域。文件数据块的查找完全由 inode 中的 i_block 数组驱动（ext4 中该数组长度为 15，每项为 32 位块号）。

1. 直接块：i_block[0] 到 i_block[11] 直接存储文件前 12 个数据块的块号。对于块大小为 4KiB 的文件系统，这覆盖 0 ~ 48KiB 的偏移。访问时只需一次磁盘读：从 inode 取块号，再读数据块。

2. 一级间接块：i_block[12] 指向一个包含块号数组的磁盘块（间接块）。该块大小即文件系统块大小（4KiB），每个块号 4 字节，因此可容纳 1024 个块号。文件最大通过一级间接访问的范围为 12 + 1024 个块，即 12 * 4K + 1024 * 4K ≈ 4.04 MiB。访问时需先读间接块，再读数据块，共两次磁盘读。

3. 二级间接块：i_block[13] 指向一个间接块，该块中的每个块号又指向一个一级间接块。因此可索引 1024 × 1024 个数据块。范围为 12 + 1024 + 1024² 个块（约 4 GiB）。访问最坏需三次磁盘读（二级间接块、一级间接块、数据块）。

4. 三级间接块：i_block[14] 指向二级间接块，形成三层间接，可索引 1024³ 个数据块。最大文件大小为 (12 + 1024 + 1024² + 1024³) × 4KiB ≈ 4 TiB（受限于块号字段和文件大小字段）。

寻址计算：给定文件偏移 offset，块大小 block_size = 4096，逻辑块号 lblock = offset / block_size。若 lblock < 12，直接使用 i_block[lblock]；若 12 ≤ lblock < 12+1024，则读取 i_block[12] 指向的间接块，取该块第 (lblock-12) 项；若 12+1024 ≤ lblock < 12+1024+1024²，则读取 i_block[13] 指向的二级间接块，先取其第 (lblock-12-1024)/1024 项得到一级间接块，再取第 (lblock-12-1024)%1024 项得到数据块号；三级间接同理，逐层向下。

与前端概念的对比：inode 的间接块寻址类似于多级页表（MMU 中虚拟地址到物理地址的转换），也与 Java/TS 中嵌套引用结构有本质区别——后者是运行时对象图遍历，而这里是磁盘上固定偏移的索引结构，每个间接块本身就是磁盘扇区，读间接块意味着一次真实 I/O。与 TS 的 union 类型对比：TS 的 union 是编译期类型展开，而 inode 的间接块是运行期按需展开的物理索引，层级深度与文件大小动态关联。前端工程师常接触的『一切皆引用』在这里不成立：块号是绝对物理地址，不是虚拟地址，没有 GC 或引用计数，删除文件只是释放 inode 和块位图。

### 3. 基础代码与实战验证
```text
验证直接/间接块寻址的最直接方法是使用 debugfs 查看真实 inode 的 i_block 内容。以下步骤无需复杂框架，在 Linux 上即可执行。

1. 创建一个 ext4 文件系统（loop 设备或临时文件）：
   truncate -s 100M /tmp/ext4.img
   mkfs.ext4 /tmp/ext4.img
   # 将镜像挂载，创建一个小文件和一个大文件，观察块映射差异
   sudo mount -o loop /tmp/ext4.img /mnt
   echo 'hello' > /mnt/small.txt   # 只占用 1 个数据块
   dd if=/dev/zero of=/mnt/large.bin bs=1M count=20   # 20MiB，触发间接块

2. 获取 inode 号：
   stat -c '%i' /mnt/small.txt   # 例如 12
   stat -c '%i' /mnt/large.bin   # 例如 13

3. 使用 debugfs 查看 inode 原始块数组：
   sudo debugfs -R 'stat <12>' /tmp/ext4.img
   # 输出中 'Blocks:' 行显示直接块号；对于小文件，所有块号都在 i_block[0] 中。

   sudo debugfs -R 'stat <13>' /tmp/ext4.img
   # 输出中会看到超过 12 个块号，第 13 个块号开始是间接块号（实际存储的是下一级块号数组）。

4. 文字化伪代码模拟块寻址（C 风格，忽略边界检查）：

   #define DIRECT_BLOCKS 12
   #define POINTERS_PER_BLOCK (4096 / 4)  // 1024

   uint32_t block_size = 4096;
   uint32_t block_number_for_offset(struct inode *inode, uint64_t offset) {
       uint32_t lblock = offset / block_size;
       uint32_t *i_block = inode->i_block;  // 15 项数组

       if (lblock < DIRECT_BLOCKS) {
           return i_block[lblock];          // 直接块：一次映射，无额外读
       }
       lblock -= DIRECT_BLOCKS;

       if (lblock < POINTERS_PER_BLOCK) {
           uint32_t *indirect = read_disk_block(i_block[12]);  // 读间接块（一次 I/O）
           return indirect[lblock];
       }
       lblock -= POINTERS_PER_BLOCK;

       if (lblock < POINTERS_PER_BLOCK * POINTERS_PER_BLOCK) {
           uint32_t *double_indirect = read_disk_block(i_block[13]);  // 读二级间接块
           uint32_t indirect_idx = lblock / POINTERS_PER_BLOCK;
           uint32_t direct_idx = lblock % POINTERS_PER_BLOCK;
           uint32_t *indirect = read_disk_block(double_indirect[indirect_idx]); // 再读一级间接块
           return indirect[direct_idx];
       }
       // 三级间接类似，不赘述
       return 0;
   }

5. 观察间接块内容：
   sudo debugfs -R 'blocks <13>' /tmp/ext4.img
   # 输出会列出所有数据块号。若用 'dump' 或 'rdump' 提取 large.bin 前几个间接块，
   # 可以看到间接块内存储的是按顺序排列的块号数组，这正是寻址机制的物证。
```

### 4. 常见误区与进阶思考
误区一：认为 ext4 所有文件都用间接块寻址。实际上 ext4 默认使用 extent 树（ext4_extent）来管理连续块，只有小文件或关闭 extent 特性（mkfs.ext4 -O ^extent）时才会使用传统间接块。但掌握间接块是理解 extent 映射的基础，因为 extent 就是针对间接块多级跳转和碎片化问题设计的：用 (起始块号 + 长度) 对代替单个块号，大幅减少索引块数量。

误区二：把 inode 中的块号当作逻辑扇区号。ext4 的块号是文件系统块号，需要乘以块大小转换为逻辑扇区（通常 512B），再经过分区偏移得到物理 LBA。此外，inode 本身也占用磁盘块（inode 表），不能与数据块混淆。

思考题：若块大小固定为 4KiB，文件偏移为 100 MiB，系统需要读取哪些磁盘块才能获取该偏移处的数据块号？请具体列出每层的间接块号（用索引值表示）以及总磁盘 I/O 次数（忽略缓存）。这需要你手动分解 lblock 的区间，并体会多级间接带来的 I/O 放大效应。
