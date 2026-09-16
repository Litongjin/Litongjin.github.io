---
title: "每日基础技术总结 · 2026-09-16 · B+树索引页分裂与合并的写放大控制"
date: 2026-09-16 08:00:00
categories: [技术分享]
tags: ["技术分享", "架构与设计"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-16 · B+树索引页分裂与合并的写放大控制

## 📚 今日主题

> **B+树索引页分裂与合并的写放大控制**（架构与设计）

### 1. 核心概念速览
1. 核心概念速览
B+树索引页分裂与合并：在 B+树中，每个节点通常映射为一个固定大小的磁盘页（如 InnoDB 16KB）。插入导致页内有序记录超出容量时，必须拆分页；删除导致页利用率过低时，可能合并或重平衡。写放大控制：以物理写字节数 / 逻辑写字节数衡量，通过分裂点选择、填充因子、合并阈值、顺序插入、WAL/刷脏批处理等手段，降低一次逻辑 INSERT/DELETE 引发的物理页写次数、随机 I/O 与 WAL 记录量。
本质：B+树用有序多路平衡树维持 O(log N) 查找；页分裂/合并是以局部写放大换取全局平衡、有序与页级 I/O 效率。它解决的是磁盘块设备上随机写代价远高于顺序写、且块大小固定，而记录长度与分布动态变化之间的矛盾。
位置：属于存储引擎核心机制，上承 SQL 执行器、事务/MVCC、索引选择，下接缓冲池、WAL/redo、doublewrite、文件系统与 SSD FTL。关系库 InnoDB、PostgreSQL、SQLite，KV WiredTiger、LMDB 等均依赖。
为什么必须掌握：后端性能问题常表现为索引维护写放大、页分裂、随机 I/O、锁等待与复制延迟。理解页分裂/合并才能正确设计主键、索引、批量导入、分区/分库分表，并在 B+树与 LSM-Tree 之间做工程选型。

### 2. 底层原理剖析
2. 底层原理剖析
页结构：页头、页目录、用户记录、空闲空间、页尾。B+树非叶子页只存 separator key + child page no；叶子页存完整行或主键+行指针，并按 key 有序，通过双向链表连接。
插入路径：
1) 从根页二分查找向下定位叶子页；
2) 若叶子页空闲空间足够，页内有序插入，更新 Page Directory，标记脏页；
3) 若不足，分配新页并分裂。InnoDB 类实现会判断插入方向：若插入 key 大于页内最大 key（顺序右插），原页保持满页，新页接收新 key，父页插入 separator；否则按约 50/50 将后半记录复制到新页。
4) 父页若满则递归分裂，直到根，根分裂时树高 +1。
合并路径：
1) DELETE 先 delete-mark，purge 后记录真正删除；
2) 若页利用率低于 MERGE_THRESHOLD（如 50%），尝试与相邻兄弟合并，移动记录并删除父页 separator；
3) 父页利用率过低则继续合并。现代引擎常延迟合并，避免抖动。
写放大来源：分裂时原页重写、新页写、父页更新、级联分裂；WAL/redo 顺序写、undo、doublewrite、binlog、刷脏与 checkpoint；随机插入 UUID 导致频繁 50/50 分裂，页利用率下降，缓存命中率降低。
控制策略：
- 主键单调递增，使插入集中在最右页，近似顺序追加；
- 方向感知分裂点，右增长时 100/0 或 90/10，随机时 50/50；
- 填充因子预留空间，降低分裂频率但增加读放大；
- 延迟合并与 merge_threshold，优先页内空间重用；
- 批量加载：排序后自底向上构建，顺序写；
- WAL 组提交、change buffer、redo 批量刷盘、doublewrite 批量；
- 监控页分裂次数、页利用率、索引碎片。
与前端对比：前端不可变更新（Immer/Immutable.js）修改树节点时复制根到叶的路径，结构共享；B+树分裂也更新父页路径，但页是固定大小磁盘块，复制成本是页级 I/O + WAL，而非对象分配。前端数组 splice 中间插入移动后续元素 O(n)，B+树将移动限制在页内并用页目录均摊，但页满分裂时页间搬移一半记录，产生写放大。React Fiber 时间切片类似缓冲池批量刷脏的批处理，但 B+树必须保证事务原子与崩溃恢复。

### 3. 基础代码与实战验证
```text
3. 基础代码与实战验证
以下 Python 极简模拟叶子页分裂与写放大差异，不依赖框架。
import bisect, random

PAGE_CAP = 8  # 模拟 16KB 页最多 8 条记录

class Page:
    def __init__(self):
        self.keys = []      # 页内有序 key，真实页中为 record + Page Directory
        self.next = None    # 叶子页链表指针

class BPlusLeafChain:
    def __init__(self):
        self.head = Page()
        self.splits = 0
        self.page_writes = 0
    def insert(self, key):
        p = self.head
        while p.next and key > p.keys[-1]:
            p = p.next   # 真实 B+树从根搜索到叶子；这里简化为叶子链定位
        if p.keys and key > p.keys[-1] and len(p.keys) >= PAGE_CAP:
            # 右端顺序插入：原页保持满页，只写新页和父页分隔键，避免 50/50 重写
            new_page = Page()
            new_page.keys.append(key)
            new_page.next = p.next
            p.next = new_page
            self.splits += 1
            self.page_writes += 2  # 新叶子页 + 父页中插入 separator key
            return
        bisect.insort(p.keys, key)  # 页内有序插入，触发记录搬移但仍在内存页内
        self.page_writes += 1       # 当前页变脏，最终刷盘一次
        if len(p.keys) > PAGE_CAP:
            mid = len(p.keys) // 2
            right = Page()
            right.keys = p.keys[mid:]   # 将后半记录复制到新页，真实分裂需同时写 WAL
            p.keys = p.keys[:mid]
            right.next = p.next
            p.next = right
            self.splits += 1
            self.page_writes += 2      # 原页重写 + 新页 + 父页更新，保守计额外 2
    def stats(self):
        pages = []
        p = self.head
        while p:
            pages.append(len(p.keys))
            p = p.next
        return self.splits, self.page_writes, pages

seq = BPlusLeafChain()
for i in range(1000):
    seq.insert(i)
print('顺序插入 splits/writes:', seq.stats()[:2])

rnd = BPlusLeafChain()
for i in random.sample(range(10000), 1000):
    rnd.insert(i)
print('随机插入 splits/writes:', rnd.stats()[:2])
print('随机页利用率:', rnd.stats()[2][:10])

验证要点：顺序插入触发右端分裂优化，原页不重写，页利用率接近 100%；随机插入频繁 50/50 分裂，页利用率低，page_writes 与 splits 更高，体现写放大。
```

### 4. 常见误区与进阶思考
4. 常见误区与进阶思考
误区一：认为页分裂只影响一个页。实际上父页可能级联分裂到根，且 WAL/undo/doublewrite/binlog 与刷脏会进一步放大；一次逻辑 INSERT 可能对应多次物理页写与 fsync。
误区二：认为删除后立即合并总是节省空间。频繁合并会造成页抖动、增加写放大与锁竞争；现代引擎通常延迟合并，优先页内空间重用，仅在利用率低于阈值时合并。
误区三：认为自增主键永远最优。自增/趋势递增可显著降低分裂，但会造成最右页写热点；分布式场景需用雪花等趋势递增 ID，避免 UUID 随机主键。
思考题：若将 B+树页填充因子从 100% 降到 70%，分裂次数减少，但读放大与缓存命中率如何变化？在什么工作负载下 70% 填充因子会降低总写放大，什么情况下反而增加？请从页利用率、缓冲池命中、刷脏频率、随机/顺序插入比例、合并阈值角度分析。
