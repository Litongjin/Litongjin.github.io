---
title: "每日基础技术总结 · 2026-09-14 · 跳跃表的层数随机化与查找/插入路径"
date: 2026-09-14 08:00:00
categories: [技术分享]
tags: ["技术分享", "算法与数据结构"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-14 · 跳跃表的层数随机化与查找/插入路径

## 📚 今日主题

> **跳跃表的层数随机化与查找/插入路径**（算法与数据结构）

### 1. 核心概念速览
跳跃表（skip list）是一种基于概率平衡的有序链表。每个节点在插入时独立随机生成高度 level，节点含 level 个前向指针 next[0..level-1]；第 i 层是第 i+1 层的子集，底层 next[0] 串接全部元素，高层是对底层的稀疏索引。层数按几何分布：P(level>=k)=p^{k-1}，通常 p=0.25 或 0.5，并设 MAX_LEVEL 截断。它解决有序结构在动态插入/删除下维持 O(log n) 查找的问题，避免红黑树/AVL 的旋转和 B+ 树的页分裂；查找/插入/删除期望 O(log n)，空间期望 O(n)，最坏 O(n) 但概率极低。位置：内存有序索引、Redis zset 的 skiplist、RocksDB/LevelDB memtable 可选结构、并发有序容器、LSM-tree 读路径。AI/后端中特征/时间戳索引、倒排索引合并、去重有序流都依赖此类结构。前端工程师必须掌握：JS 的 Map/Set 期望 O(1) 但无序，Array.sort+二分查找 O(log n) 但插入 O(n)；跳表用随机化把链表改造成可二分下降的多层索引，是理解概率数据结构、无锁并发和存储引擎索引的底层基石。

### 2. 底层原理剖析
底层结构：head 固定持有 MAX_LEVEL 个前向指针，currentLevel 表示当前实际最高层。查找路径从 currentLevel-1 层开始，每层尽量向右移动，直到 next 为 NIL 或 next.key >= target，然后下降一层；第 0 层得到候选。伪代码：
x = head
for i = currentLevel-1 downto 0:
  while x.next[i] != NIL and x.next[i].key < target:
    x = x.next[i]
candidate = x.next[0]
if candidate != NIL and candidate.key == target: found
else: not found（x 即第 0 层插入前驱）。
本质：高层做粗粒度跳跃，底层做细粒度线性推进；每层期望检查 1/p 个节点，层数期望 log_{1/p} n，故总期望 O(log n)。
层数随机化：randomLevel(): level=1; while random()<p and level<MAX_LEVEL: level+=1; return level。P(level=k)=p^{k-1}(1-p)，P(level>=k)=p^{k-1}；期望高度 1/(1-p)，期望指针数 1/(1-p)。随机性与 key 无关，因此不依赖输入顺序，不需要重平衡。
插入路径：先执行查找路径，同时用 update[0..MAX_LEVEL-1] 记录每层最后一个 key 小于新 key 的节点。然后 lvl=randomLevel()；若 lvl>currentLevel，则对 i=currentLevel..lvl-1 令 update[i]=head，并 currentLevel=lvl。创建 node(key,lvl)，对 i=0..lvl-1：node.next[i]=update[i].next[i]; update[i].next[i]=node。顺序必须是先接后继再改前驱，否则断链。删除同理，先查找 update，若命中则逐层改指针，必要时降低 currentLevel。
与前端已有概念对比：JS 数组是连续内存，二分查找 O(log n) 但 splice 插入/删除 O(n) 搬移；Map/Set 基于哈希，平均 O(1) 查找但无序，范围查询需全量扫描或额外排序。跳表是链表+多级稀疏索引，动态插入 O(log n) 期望，天然支持有序遍历和范围查询。与 Java 接口/TS 接口的异同：Java 接口是运行时类型契约（方法表、invokeinterface），TS 接口是编译期结构类型，运行时无痕迹；它们都是抽象边界。跳表的层不是类型契约，而是运行时物理索引，随机生成且影响数据分布；共同点是都通过抽象隐藏实现细节，但跳表隐藏的是平衡维护，接口隐藏的是调用约定。与 B+ 树对比：B+ 树页式结构磁盘友好，跳表指针式结构内存/并发友好，范围查询都沿底层有序链表/叶子链。

### 3. 基础代码与实战验证
```text
极简 Python 验证：
import random

MAX_LEVEL = 16
P = 0.25

class Node:
    def __init__(self, key, level):
        self.key = key
        self.next = [None] * level  # next[i] 指向同层后继；level 即节点高度

class SkipList:
    def __init__(self):
        self.head = Node(None, MAX_LEVEL)  # head 不存业务 key，固定最高层
        self.level = 1  # 当前实际最高层，至少为 1

    def random_level(self):
        lvl = 1
        while random.random() < P and lvl < MAX_LEVEL:
            lvl += 1  # 每次以 P 概率晋升，形成几何分布
        return lvl

    def search(self, key):
        x = self.head
        for i in range(self.level - 1, -1, -1):
            while x.next[i] and x.next[i].key < key:
                x = x.next[i]  # 本层尽可能右移，直到越过目标或到 NIL
        x = x.next[0]  # 下降至第 0 层后的候选节点
        if x and x.key == key:
            return True
        return False

    def insert(self, key):
        update = [None] * MAX_LEVEL  # update[i] 记录第 i 层插入位置的前驱
        x = self.head
        for i in range(self.level - 1, -1, -1):
            while x.next[i] and x.next[i].key < key:
                x = x.next[i]  # 查找路径同时记录每层前驱
            update[i] = x
        lvl = self.random_level()  # 新节点层数在插入时一次性随机确定
        if lvl > self.level:
            for i in range(self.level, lvl):
                update[i] = self.head  # 新增高层的前驱只能是 head
            self.level = lvl
        node = Node(key, lvl)
        for i in range(lvl):
            node.next[i] = update[i].next[i]  # 先接后继
            update[i].next[i] = node  # 再改前驱，顺序反了会丢失后续链
        return True

验证：依次插入 1..20，调用 search 检查命中与不命中；观察 self.level 与随机层数。若要观察路径，可在 while 中打印 i 和 x.key，确认每层只做局部右移后下降，而非从底层线性扫描。
```

### 4. 常见误区与进阶思考
误区一：把随机化理解为性能会随机退化。跳表最坏确实 O(n)，但层数独立同分布，P(level>=k)=p^{k-1}，查找期望 O(log n)，空间期望 O(n)。工程上设 MAX_LEVEL 并用固定 p，可把退化概率压到可忽略；但硬实时/确定性最坏界场景仍应选红黑树或 B+ 树。
误区二：认为查找或后续操作会重新随机层数，或插入高节点后需要提升已有节点。层数只在插入时生成一次并写入节点结构，之后不变；查找只沿已有层下降，不进行任何随机。插入只修改 update 数组指向的前驱/后继指针，已有节点的层数不变。另一个实现错误是更新指针顺序反了，导致后继链断裂。
思考题：把晋升概率 p 从 0.5 改为 0.25，期望层数、每节点期望指针数、每层水平比较次数和总查找常数如何变化？为什么 Redis zskiplist 采用 p=0.25、MAX_LEVEL=32，而不是 p=0.5？请从空间/时间权衡、可覆盖元素规模 P(level>=32)=0.25^31、以及缓存局部性角度推导。
