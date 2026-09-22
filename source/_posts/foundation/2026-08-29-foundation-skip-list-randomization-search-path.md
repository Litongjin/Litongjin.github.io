---
title: "每日基础技术总结 · 2026-08-29 · 跳跃表的层数随机化与查找/插入路径"
date: 2026-08-29 08:00:00
categories: [技术分享]
tags: ["技术分享", "算法与数据结构"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-29 · 跳跃表的层数随机化与查找/插入路径

## 📚 今日主题

> **跳跃表的层数随机化与查找/插入路径**（算法与数据结构）

### 1. 核心概念速览
跳跃表（Skip List）是有序链表上的概率化多级索引结构。每个节点包含一个 forward 指针数组，层数由独立随机过程决定；底层是完整有序链表，上层是稀疏索引。核心机制是：以几何分布随机选取节点进入更高层，使每一层节点数约为下一层的 p 倍，从而在有序链表上构造出期望的二分查找路径。它解决的是普通有序链表顺序查找 O(n) 的问题，同时避免了 AVL/红黑树为实现严格平衡需要的旋转与父指针维护，用随机化换取 O(log n) 的期望查找/插入/删除复杂度，且支持有序范围遍历。在计算机体系中，它位于内存索引层的核心位置：Redis 的 Sorted Set、LevelDB/RocksDB 的 MemTable 都以此作为有序索引。专业工程师必须掌握，因为它是概率数据结构、分布式有序存储和 NoSQL 内核的共同基础，能从根本上理解『用冗余空间换时间』与『随机化平衡』的工程取舍。

### 2. 底层原理剖析
层数随机化的本质是几何分布。令底层为第 0 层，节点的第 k 层（k≥0）指针存在的概率为 p^k；等价地，层数 L（从 1 计）满足 P(L=k)=p^{k-1}(1-p)。每次‘升层’是一次伯努利试验：从 0 层开始，以概率 p 增高一层，直到失败。因此层数期望为 1/(1-p)，节点高度以指数概率衰减，整体最高层约为 log_{1/p} n，这是 O(log n) 路径长度的概率根源。查找路径是从 header 的当前最高层开始，每层从左向右移动，直到下一个节点值大于等于目标，然后降层；重复到第 0 层。因为每层节点数量按 p 递减，搜索路径呈现‘先向右跳跃，再向下深入’的锯齿形，比较次数期望为 O(log_{1/p} n)。插入路径与查找路径共享：先在每一层记录最后一个小于 value 的节点到 update[]，然后随机生成新节点层数 lvl；若 lvl 大于当前最高层，需要把新增层的 update 值设为 head，再把新节点按 update 指针串入每一层。与前端已有概念对比：TypeScript 的 interface 是编译期约束，编译后完全擦除，不参与运行时内存布局；跳跃表的层是运行期真实存在的指针数组，不是类型而是物理冗余索引。前端常用的 Map/对象哈希查找是 O(1) 但无序，有序数组可二分但插入是 O(n)；跳跃表用概率冗余指针同时获得近似二分查找的路径和链表式有序插入，本质上是把‘随机化’作为平衡手段，而不是像红黑树那样用旋转维护确定性不变量。

### 3. 基础代码与实战验证
```text
const MAX_LEVEL = 16;

class Node {
  constructor(val, level) {
    this.val = val;
    this.next = new Array(level + 1).fill(null); // next[i] 是当前节点在第 i 层的后继
  }
}

class SkipList {
  constructor() {
    this.head = new Node(-Infinity, MAX_LEVEL);
    this.level = 0; // 当前实际最高层索引，0 表示只有底层
  }

  randomLevel() {
    let lvl = 0;
    // 每次以 0.5 概率升高一层，最终层数服从几何分布，期望层数=2
    while (lvl < MAX_LEVEL && Math.random() < 0.5) lvl++;
    return lvl;
  }

  insert(val) {
    const update = new Array(MAX_LEVEL + 1); // update[i]：在第 i 层最后一个小于 val 的节点
    let cur = this.head;

    // 从最高层逐层向下，记录各层前驱；这是找到插入位置的核心路径
    for (let i = this.level; i >= 0; i--) {
      while (cur.next[i] && cur.next[i].val < val) cur = cur.next[i];
      update[i] = cur;
    }

    const lvl = this.randomLevel();

    // 如果新节点层数超过当前最高层，则新增层的前驱必须是 head
    if (lvl > this.level) {
      for (let i = this.level + 1; i <= lvl; i++) update[i] = this.head;
      this.level = lvl;
    }

    const node = new Node(val, lvl);
    // 在每一层执行链表插入：把新节点串进第 i 层的前驱之后
    for (let i = 0; i <= lvl; i++) {
      node.next[i] = update[i].next[i];
      update[i].next[i] = node;
    }
  }

  contains(val) {
    let cur = this.head;
    // 查找与插入共享路径：每层向右逼近，再下降到底层
    for (let i = this.level; i >= 0; i--) {
      while (cur.next[i] && cur.next[i].val < val) cur = cur.next[i];
    }
    const target = cur.next[0];
    return target !== null && target.val === val;
  }
}

// 验证：插入元素后，底层完整链表保留所有有序节点，上层是概率索引
const sl = new SkipList();
[3, 7, 1, 9, 4].forEach(v => sl.insert(v));
console.log(sl.contains(7)); // true
console.log(sl.contains(5)); // false
```

### 4. 常见误区与进阶思考
误区一：认为随机化会让结构‘不稳定’，于是试图用确定性算法强制层数或加入旋转调整。这是方向错误。跳跃表的随机化本身就是平衡手段：层数服从几何分布，整体高度以高概率维持在 O(log n)，不需要全局调整；单次随机波动在样本量增大后被统计规律淹没。误区二：插入时只记录当前层 update，忽略新节点层数超过当前最高层的处理。如果 lvl > this.level，必须把 this.level+1 到 lvl 的所有 update[i] 指向 head，否则新层的 next 指针会指向 null 或旧数据，查找路径从最高层出发时会直接漏掉节点。思考题：把 randomLevel 的升层概率 p 从 0.5 改为 0.25，期望层数、每层节点密度、查找路径长度和内存占用分别如何变化？为什么 Redis 的 zslRandomLevel 使用 p=0.25 而不是 0.5？这决定了你是否真正理解层数随机化与时间复杂度/空间开销的精确权衡。
