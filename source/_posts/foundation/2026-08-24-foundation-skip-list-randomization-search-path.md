---
title: "每日基础技术总结 · 2026-08-24 · 跳跃表的层数随机化与查找/插入路径"
date: 2026-08-24 08:00:00
categories: [技术分享]
tags: ["技术分享", "算法与数据结构"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-24 · 跳跃表的层数随机化与查找/插入路径

## 📚 今日主题

> **跳跃表的层数随机化与查找/插入路径**（算法与数据结构）

### 1. 核心概念速览
跳跃表（Skip List）是一种基于有序链表、通过随机化层数构造多层索引的概率性数据结构。其核心机制：每个节点在插入时按几何分布随机决定自己的层数，底层是全量有序链表，上层是逐层稀疏的索引；查找时从最高层开始，沿着「同层前进、层间下降」的路径逼近目标，使查找、插入、删除的期望时间复杂度均为 O(log n)，空间期望 O(n)。它解决的本质问题是在保持链表指针插入/删除灵活性的同时，消除有序链表线性查找的瓶颈。在计算机体系结构中，它处于基础数据结构层，是 Redis 有序集合、LevelDB/RocksDB 的 MemTable、Apache HBase 等系统的核心实现；专业工程师掌握它是为了理解概率算法如何在工程中替代复杂平衡树，以及如何在高并发/持久化场景下权衡内存与速度。

### 2. 底层原理剖析
层数随机化的本质：设底层为第 0 层，每个节点出现在第 i 层的概率为 p^i（p∈(0,1)），层数 L 满足 P(L=k)=p^k(1-p)。这样第 i 层的节点数期望为 n·p^i，形成从下到上指数递减的稀疏索引。查找时，从当前最高层 this.level 开始，每层内按序前进直到后继 key 不小于目标，然后下降到下一层继续；到达第 0 层后，后继即为不小于目标的最小节点。整个路径长度期望为 O(log_{1/p} n)，因为每下降一层都会剪掉一部分空间，类似二分查找。插入路径与查找路径基本相同，但需要维护 update 数组记录每一层最后一个 key 小于目标的前驱节点，然后按随机生成的 lvl 创建新节点，并从第 0 层到第 lvl 层把新节点链接进 update[i] 的 next。由于层数随机，插入时无需像平衡树那样旋转或染色，而是用概率换取了结构自平衡。
与前端已有概念的对比：可以把跳跃表类比为「多层单向链表的二分查找」，而前端熟悉的数组二分查找是「连续内存的确定性折半」。两者都实现 O(log n) 查找，但前者通过额外指针而非物理下标实现跳跃，后者需要随机访问能力。这种差异类似于 Java 接口与 TS 接口：二者都定义契约，但 Java 接口是运行时/编译期的名义类型，TS 接口是编译期的结构类型，背后机制不同。理解跳跃表，就是理解在链式结构上如何用概率构造索引，以突破线性遍历。

### 3. 基础代码与实战验证
```text
class SkipListNode {
  constructor(key, value, level) {
    this.key = key;
    this.value = value;
    // next[i] 指向第 i 层的后继节点；level 越大，索引越稀疏
    this.next = new Array(level + 1);
  }
}

class SkipList {
  constructor(p = 0.5, maxLevel = 16) {
    this.p = p;           // 晋升概率：每次有 p 的概率多一层
    this.maxLevel = maxLevel; // 层数上限，防止随机数极端导致空间爆炸
    this.level = 0;       // 当前实际最高层（从 0 开始）
    this.header = new SkipListNode(-Infinity, null, maxLevel); // 哨兵头节点
  }

  randomLevel() {
    let lvl = 0;
    // 以概率 p 递增层数，产生几何分布：P(lvl = k) = p^k * (1-p)
    while (Math.random() < this.p && lvl < this.maxLevel) {
      lvl++;
    }
    return lvl;
  }

  find(key) {
    let curr = this.header;
    // 从最高层开始，向下逐层搜索
    for (let i = this.level; i >= 0; i--) {
      // 在同一层中，只要后继节点的 key 小于目标，就前移
      while (curr.next[i] && curr.next[i].key < key) {
        curr = curr.next[i];
      }
      // 当前层无法继续前进，则下降到下一层
    }
    curr = curr.next[0]; // 底层中第一个大于等于 key 的节点
    return curr && curr.key === key ? curr : null;
  }

  insert(key, value) {
    const update = new Array(this.maxLevel + 1); // 记录每一层待更新的前驱节点
    let curr = this.header;
    for (let i = this.level; i >= 0; i--) {
      while (curr.next[i] && curr.next[i].key < key) {
        curr = curr.next[i];
      }
      update[i] = curr; // 第 i 层需要更新链接的节点
    }
    const lvl = this.randomLevel();
    if (lvl > this.level) {
      // 新层的前驱是 header；同时提升当前实际层高
      for (let i = this.level + 1; i <= lvl; i++) {
        update[i] = this.header;
      }
      this.level = lvl;
    }
    const newNode = new SkipListNode(key, value, lvl);
    // 逐层将新节点插入到 update[i] 之后
    for (let i = 0; i <= lvl; i++) {
      newNode.next[i] = update[i].next[i];
      update[i].next[i] = newNode;
    }
  }
}
```

### 4. 常见误区与进阶思考
常见误区：
1. 认为跳跃表是「碰运气」的数据结构，担心最坏情况 O(n)，不如平衡树可靠。实际上，跳跃表的复杂度是概率性保证：当随机源质量良好时，退化的概率随 n 增大指数级下降；工程中通过设置合适的 p 和 maxLevel（如 Redis 的 p=1/4, maxLevel=32）来平衡内存与速度。它不是不严谨，而是用随机化换取了实现的简洁性。
2. 实现插入时忽略新层 update 的初始化。当随机层数 lvl 超过当前 this.level 时，如果不把 update[i]（i > this.level）指向 header，也没有提升 this.level，新节点的高层指针会悬空，导致后续查找无法访问高层索引，结构退化。这个 bug 在面试或生产代码中极常见。

进阶思考题：如果将随机层数改为固定常数（如每个节点都是 4 层），跳跃表会退化成什么结构？查找复杂度是多少？请用层数分布和搜索路径长度解释。
