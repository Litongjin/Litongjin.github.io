---
title: "每日基础技术总结 · 2026-08-26 · B+树索引页分裂与合并的写放大控制"
date: 2026-08-26 08:00:00
categories: [技术分享]
tags: ["技术分享", "架构与设计"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-26 · B+树索引页分裂与合并的写放大控制

## 📚 今日主题

> **B+树索引页分裂与合并的写放大控制**（架构与设计）

### 1. 核心概念速览
B+树索引页分裂与合并是存储引擎在索引页容量溢出（插入路径）或下溢（删除路径）时，对页内键值进行重新分布以维持B+树结构不变量（所有叶子页等深、非根页填充率不低于50%、父页分隔键严格覆盖子页键值域）的页重组机制。本质：插入使页内键数达到N+1时，取中点将键集切分为两个约N/2的页，并将分隔键上提至父页；删除使页内键数低于N/2时，与相邻兄弟页执行合并（两页键数合计不超过N）或再平衡（借键，仅更新父页分隔键）。写放大（Write Amplification）指一次逻辑写入在物理存储层实际触发的页写入次数或字节数；页分裂/合并是B+树写放大的核心来源——一次叶子页分裂至少产生左页整页重写、右页新页分配、父页分隔键更新共3次页写，若父页满则级联向上传播，最坏触发根分裂使树高+1。该机制解决的核心矛盾：在保证O(logN)点查与范围扫描性能、维持页空间利用率的前提下，最小化随机写负载下的额外物理写入，直接决定数据库写入吞吐、索引碎片率与SSD寿命。体系位置：位于存储引擎索引层，是磁盘IO调度、缓冲池（Buffer Pool）与事务日志（redo/undo）三者的交汇点，是OLTP写入路径的瓶颈枢纽；同时也是LSM-tree等替代结构的写放大对照基准，以及分布式数据库分片、向量索引（HNSW/IVF）节点重组设计的底层参考。专业工程师必须掌握：因为索引碎片治理、批量插入性能、删除后空间回收、自增主键为何更优等日常决策的底层依据全部由分裂/合并行为决定；前端工程师的内存数组操作直觉在此完全失效——页是磁盘最小IO单位，修改1字节也必须整页读改写。

### 2. 底层原理剖析
1. 页容量约束与分裂触发机制
每个非根页键数被约束在 [ceil(N/2), N] 区间。插入路径自根向下定位叶子页，若目标页已有N个键，新键加入后为N+1个，违反上限，触发分裂。分裂点取 mid = (N+1)//2：左页保留前mid个键，右页获得剩余键，右页最小键作为分隔键上提至父页，父页新增一条(key, right_pointer)。若父页此时也满，递归分裂，直至根；根分裂时创建新根，树高+1。

2. 分裂的写放大构成
以16KB页为例，一次叶子分裂的页写清单：左页因键集改变需整页重写（16KB）；右页为新分配页需写入（16KB）；父页因插入分隔键需重写（16KB）；若父页满则祖父页再写，级联k层产生约2k+1次页写。此外，redo log须记录分裂产生的逻辑变更（物理逻辑日志，非整页镜像）；页落盘时若开启doublewrite，每个脏页还要先写doublewrite buffer再写实际位置，物理写放大再乘2。

3. 删除、下溢与合并/再平衡
删除使页键数降至ceil(N/2)-1时下溢。此时先检查左/右兄弟页：若本页与兄弟键数之和不超过N，执行合并——两页键合并至左页，右页释放回空闲列表，父页删除对应分隔键；若父页因此下溢则递归合并。若两页键数之和大于N，执行再平衡：从兄弟页借出若干键，使两页均恢复不低于N/2，父页仅更新分隔键，不删除键、不释放页。

4. 写放大控制的四个核心机制
（1）非对称阈值：分裂阈值N与合并阈值N/2之间的滞后区间抑制分裂-合并振荡，避免同一页在临界点反复重组；（2）顺序写优化：连续插入时分裂恒发生在最右页，新页通过AIO顺序追加，随机IO被转化为顺序IO；（3）预分裂/预合并：插入路径上预检兄弟页状态，提前分流，降低递归分裂概率；（4）合并延迟：前台删除仅做删除标记，页合并由后台purge线程按需执行，把写放大从关键路径剥离。

5. 叶子页分裂/合并核心伪代码
insert(page, key):
    page.keys.append(key); sort(page.keys)
    if len(page.keys) > N:
        mid = len(page.keys) // 2
        right = new_page(); right.keys = page.keys[mid:]
        page.keys = page.keys[:mid]
        parent.insert(right.keys[0], right)   # 可能级联分裂
delete(page, key):
    page.keys.remove(key)
    if len(page.keys) < N // 2:
        sib = find_sibling(page)
        if len(page.keys) + len(sib.keys) <= N:
            merge(page, sib)                  # 父页删除分隔键，可能级联合并
        else:
            rebalance(page, sib)              # 借键，父页仅更新分隔键

6. 与前端概念的对比：Java接口 vs TS接口
TS接口是编译期结构类型约束，运行时被擦除，零成本；Java接口是运行期多态契约，经虚方法表分派，存在运行期查找成本。B+树的分裂/合并协议更接近Java接口的运行期强制力——页间键序、填充率、父子指针一致性是磁盘上持续生效的物理不变量，任何违反都会导致索引损坏；但维护这些不变量的每次操作都直接产生物理页写入。前端工程师易以TS思维（形状对了就行）理解数据结构，忽略物理页填充率与整页IO约束。二者差异的本质：TS/Java接口约束的是内存中的逻辑形状，B+树约束的是磁盘上的物理分布，后者每次维护都有真实IO代价。

### 3. 基础代码与实战验证
```text
以下为不依赖任何框架的Python极简模拟，验证页分裂/合并的写放大计数机制：

class Leaf:
    def __init__(self, cap):
        self.cap = cap          # 页容量（最大键数）
        self.keys = []          # 有序键列表，模拟页内记录
        self.next = None        # 右兄弟指针（叶子页链表）

    def insert(self, key):
        # 就地插入并保持有序：一次页修改在存储层等价于一次整页写
        self.keys.append(key)
        self.keys.sort()
        if len(self.keys) > self.cap:
            # 页满分裂：把N+1个键对半切分，保证两侧均不低于半满
            mid = len(self.keys) // 2
            right = Leaf(self.cap)
            right.keys = self.keys[mid:]   # 右页获得较大的半区
            self.keys = self.keys[:mid]    # 左页保留较小的半区
            return right                   # 返回新右页
        return None

class LeafList:
    def __init__(self, cap):
        self.head = Leaf(cap)
        self.wa = 0              # 累计写放大（单位：页写次数）

    def insert(self, key):
        node = self.head
        while node.next and node.keys[-1] < key:
            node = node.next     # 顺序定位目标页（真实B+树为O(logN)查找）
        right = node.insert(key)
        if right is not None:
            self.wa += 2         # 左页整页重写1次 + 右页新页写入1次
            right.next = node.next
            node.next = right
            self.wa += 1         # 父页插入分隔键并更新指针：第3次页写
        else:
            self.wa += 1         # 未分裂：仅目标页一次写

    def delete(self, key):
        prev = None
        node = self.head
        while node and key not in node.keys:
            prev = node
            node = node.next
        if node is None:
            return
        node.keys.remove(key)
        self.wa += 1             # 删除也是一次页修改
        # 下溢判定：键数低于半满且与左兄弟合计不超过一页容量 → 合并
        if len(node.keys) < node.cap // 2 and prev is not None and len(prev.keys) + len(node.keys) <= node.cap:
            prev.keys.extend(node.keys)
            prev.keys.sort()
            prev.next = node.next
            self.wa += 2         # 左页重写1次 + 父页删除分隔键1次

import random
random.seed(42)
lt = LeafList(16)
for _ in range(2000):
    lt.insert(random.randint(0, 10**6))
print('2000次随机插入累计WA(页写次数):', lt.wa)
print('理论下限(每键一次页写): 2000')
print('写放大倍数:', lt.wa / 2000)
for _ in range(500):
    lt.delete(random.randint(0, 10**6))
print('再执行500次随机删除后累计WA:', lt.wa)

关键注释：
- 'mid = len(self.keys) // 2'：分裂点取中点，维持每页至少50%填充率，这是B+树空间利用率的底线；
- 'self.wa += 2'：一次分裂至少产生2次页写（左页重写+右页分配），是写放大的直接来源；
- 'self.wa += 1'（父页）：真实引擎中父页更新可能级联触发祖父页分裂，最坏沿树高传播，写放大随树高倍增；
- 'len(prev.keys) + len(node.keys) <= node.cap'：这是合并与再平衡的分界条件；超过容量则必须借键（再平衡），仅更新父页分隔键、不释放页。真实引擎的合并会沿父节点递归向上，本模拟只演示单层合并的写放大计数。
```

### 4. 常见误区与进阶思考
误区1：把页分裂理解为只影响当前页。实际一次叶子分裂至少产生3次页写（左页重写+右页新写+父页更新），且父页满时级联传播，最坏沿树高逐层分裂；若再计入redo log与doublewrite，一次逻辑插入的物理写放大可达10倍以上。前端出身者容易把内存中改一个数组元素O(1)的成本直觉迁移到数据库，但磁盘页是整页读写的最小单位，页内1字节的变更也需重写整页——这正是B+树写放大区别于前端内存操作的物理根源。

误区2：认为合并是分裂的逆操作、阈值对称。分裂在页满（N）触发，合并却要在页使用率低于N/2且与兄弟合计不超过一页容量时才触发，中间存在N/2到N的滞后区。若对称设计（都在50%触发），交替插入/删除会让同一页反复分裂-合并，产生振荡式写放大。此外，合并与再平衡是两种不同操作：再平衡不释放页、只改父页分隔键；合并释放页、删除父键，并可能引发父页下溢的递归合并。

思考题：InnoDB页大小16KB，开启doublewrite，buffer pool中目标叶子页与父页均为脏页，redo采用物理逻辑日志。一次二级索引叶子页分裂（父页未满、无级联）从发起插入到两个页最终落盘，SSD上物理写入字节数的下限与上限分别是多少？请逐项推导：左页重写、右页分配、父页更新、redo日志块写入（512B对齐/group commit聚合）、doublewrite对每个脏页的额外16KB、以及脏页刷盘时机对重复写的影响。注意：下限绝不是3×16KB=48KB——doublewrite使每个脏页落盘写两次；上限则需考虑日志刷盘与页刷盘的交错重复写。这个推导能区分逻辑页写放大与物理字节写放大两个层次。
