---
title: "每日基础技术总结 · 2026-09-05 · B+树索引页分裂与合并的写放大控制"
date: 2026-09-05 08:00:00
categories: [技术分享]
tags: ["技术分享", "架构与设计"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-05 · B+树索引页分裂与合并的写放大控制

## 📚 今日主题

> **B+树索引页分裂与合并的写放大控制**（架构与设计）

### 1. 核心概念速览
B+树索引页分裂与合并的写放大控制，是存储引擎为降低“逻辑操作引发的实际物理写入量”而设计的一套阈值与缓冲机制。其本质是：当插入/删除使页（page）容量越过边界时，B+树必须通过页分裂/页合并来维持平衡；每一次分裂或合并都会改写当前页、兄弟页以及祖先路径上的多层中间页，并可能新建或释放页。逻辑上一条记录的操作，物理上可能对应多次16KB页的写回，这就是写放大。控制写放大的核心不是消除结构维护（那会使树退化为不平衡），而是让分裂/合并发生的概率与代价受控：预留空闲空间、设置惰性合并阈值、通过buffer pool与change buffer延迟并批量落盘。该机制位于数据库内核的存储引擎层，介于缓冲管理器与磁盘I/O之间；专业工程师必须掌握它，因为它是分析事务吞吐、SSD磨损、后台刷新、索引碎片化与锁竞争的共同底层基础。

### 2. 底层原理剖析
B+树以固定大小的页作为I/O单位。页内记录按主键有序，中间节点存放键与孩子指针。插入路径：先定位叶节点；若叶页空间不足以容纳新记录，则分配一个新页，将约一半记录迁移到新页，再把新页的最小键与指针插入父节点；若父节点也满，则递归分裂，直到根节点，形成一个新根。删除路径：记录删除后若叶页占用率低于合并阈值，优先从相邻兄弟页“借”记录（重分布）；若兄弟也没有富余，则把两个页合并为一个，删除父节点中的对应指针，递归处理父节点下溢。

写放大来源有四个层次：1. 分裂/合并本身会产生新页或使一个页变为空页；2. 当前页的剩余记录要重新写回；3. 父节点更新，甚至一直级联到根；4. 被释放的页在后续分配时也可能需要初始化写。因此一次单记录写入，物理上可能产生O(log n)次页写。

控制机制的核心是“阈值滞回”和“写缓冲”。填充因子（fill factor）在构建索引时预留每页空闲空间，插入时不会立即溢出，从而降低分裂频率。合并阈值通常设置为低于分裂产生的平均占用率（如InnoDB的MERGE_THRESHOLD默认50%），使页状态在分裂后落在“分裂阈值”与“合并阈值”之间的缓冲带内，避免边界记录刚分裂又合并的抖动。change buffer将非唯一辅助索引的删除/插入操作暂存在内存中，按页面批量应用，把多次随机小写合并成一次顺序页改写。buffer pool的延迟刷盘则让多次逻辑操作产生的脏页在checkpoint时合并成一次物理写，这是宏观层面的写放大收敛。

与前端已有概念的对比：这类似于React把多次setState合并成一次commit，而不是每次立即操作DOM；也类似于Java/C++容器为vector预留capacity从而减少扩容时的批量拷贝。区别在于：B+树面对的是持久化介质，合并和批量化必须以可崩溃恢复为边界，任何合并后的数据都要能通过redo/undo重建；而前端batched updates没有持久化约束，可以只保留最终状态。

### 3. 基础代码与实战验证
```text
class BPlusPage:
    CAPACITY = 8                       # 每页最多记录数，抽象表示16KB物理页容量
    MERGE_THRESHOLD = 0.5              # 合并阈值：低于50%才考虑合并
    FILL_FACTOR = 0.7                  # 预留30%空闲空间，降低随机插入时的分裂概率

    def __init__(self):
        self.records = []              # 叶页记录；内部节点则存键和孩子指针

    def occupancy(self):
        return len(self.records) / self.CAPACITY

    def split(self):
        # 分裂点取中点，使左右两页占用率约50%
        mid = len(self.records) // 2
        right = BPlusPage()
        right.records = self.records[mid:]
        self.records = self.records[:mid]
        return right

    def can_insert(self):
        # 在超过fill_factor限制前允许原位插入，推迟分裂发生
        return len(self.records) + 1 <= self.CAPACITY * self.FILL_FACTOR


def insert(leaf, rec, parent):
    if leaf.can_insert():
        leaf.records.append(rec)       # 只修改内存页并标记dirty，物理写交给buffer pool统一flush
        return

    right = leaf.split()               # 记录数已到阈值，必须分裂
    # 此时leaf和right各约50%占用，远离merge_threshold，避免刚分裂就合并
    insert_into_parent(parent, leaf, right)
    # 若parent也满，则递归分裂；每次递归都导致额外一次页写


def delete(leaf, key):
    leaf.records.remove(key)
    if leaf.occupancy() < leaf.MERGE_THRESHOLD:
        sibling = left_sibling(leaf)
        if sibling and sibling.occupancy() > leaf.MERGE_THRESHOLD:
            redistribute(leaf, sibling)          # 先借记录，避免页合并的更高写代价
        else:
            merge(leaf, sibling)                 # 合并后要删除父页中的指针，可能级联
            # 合并阈值低于分裂阈值，形成滞回区间；否则会反复分裂/合并造成写放大
```

### 4. 常见误区与进阶思考
1. 误区一：认为“页满就分裂，页低于50%就合并”是最优策略。实际必须保证分裂阈值与合并阈值之间存在滞回带。如果把合并阈值设置过高甚至等于分裂后目标占用率，那么边界位置的一次删除会立即触发合并，随后一次插入又触发分裂，导致原本只需1次I/O的删除+插入变成2~3次页写入。正确的认知是：分裂后的页占用率必须落在“分裂触发点”和“合并触发点”之间，系统才能稳定。
2. 误区二：只看到单次分裂新增了一个页，忽略写放大是由“逻辑操作次数 × 级联页数 × 物理刷盘批次”三者共同决定。buffer pool延迟刷盘会让相邻多次分裂在checkpoint时合并为同一次物理写；change buffer则把辅助索引的多次随机写合并为对目标页的批量应用。因此调优时不能只调fill factor，必须结合redo log group commit、后台刷页速率和change buffer覆盖范围综合判断。

思考题：某页在INSERT达到阈值后被50/50分裂，两个子页占用率约为50%。随后同一主键位置上连续发生DELETE 1条、INSERT 1条。请问：若把MERGE_THRESHOLD设为0.5，页面层会发生什么？若设为0.4呢？两种设置下的写放大差异本质是什么？
