---
title: "每日基础技术总结 · 2026-09-03 · 跳跃表的层数随机化与查找/插入路径"
date: 2026-09-03 08:00:00
categories: [技术分享]
tags: ["技术分享", "算法与数据结构"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-03 · 跳跃表的层数随机化与查找/插入路径

## 📚 今日主题

> **跳跃表的层数随机化与查找/插入路径**（算法与数据结构）

### 1. 核心概念速览
跳跃表（Skip List）是一种基于有序链表的多层级索引数据结构。每个节点维护一组指向不同距离后继节点的指针（forward），层数由随机过程决定。其本质是用概率平衡代替确定性平衡（如红黑树），使得查找、插入、删除的期望时间复杂度均为 O(log n)，空间复杂度为 O(n)。它解决的问题是：在有序动态数据集上既要支持高效的查找，又要支持低成本的插入/删除，且避免平衡树旋转的复杂实现。在计算机体系中的位置：跳跃表是基础数据结构，被 Redis 等数据库/缓存系统用作有序集合的底层实现；也是理解概率数据结构（如布隆过滤器、LSM-Tree）的基石。专业工程师必须掌握它，因为实际系统会用到，且它展示了随机化如何优雅地解决确定性问题，这对设计高并发、低延迟系统至关重要。

### 2. 底层原理剖析
层数随机化机制：插入节点时，从第 1 层开始，以概率 p（通常 p=0.5）决定是否进入下一层，直至失败或达到最大层数。因此某节点恰好有 k 层的概率为 p^(k-1)*(1-p)，期望层数为 1/(1-p)。高层节点按指数递减，从而使搜索路径期望长度呈对数级。

查找路径：从 header 的最高有效层（level）出发，在当前层不断向右移动，直到下一个节点的值大于等于目标值；然后下降到下一层，重复此过程。最终在第 0 层定位到目标或确认不存在。本质上是将链表的顺序遍历转化为多级跳跃，类似二分查找的分治思想，但无需随机访问。

插入路径：先执行一次查找，同时记录每一层中最后一个小于待插入值的节点，即 update 数组。然后调用随机层数生成函数获得 lvl。若 lvl > 当前 level，则初始化新层的前驱为 header，并更新 level。最后从第 0 层到 lvl-1 层，将新节点逐层链入。删除路径同理，需要记录每层待删除节点的前驱并重连。

与前端概念的对比：前端工程师熟悉数组和哈希表。数组支持 O(1) 随机访问但插入/删除 O(n)；哈希表查找 O(1) 但无序。跳跃表牺牲了内存空间（额外指针）换取有序性和 O(log n) 动态操作。其随机化与哈希表的冲突处理一样，都是概率化设计；但与哈希表不同，跳跃表保持元素的自然顺序。类似地，Java 接口是运行时抽象类型，TS 接口是编译期结构契约，二者机制不同但目的类似——跳跃表和平衡树都提供有序集合，但跳跃表通过随机层数隐式维护平衡，而平衡树通过旋转操作显式维护平衡，二者在实现简单性、缓存局部性、并发控制上各有差异。

### 3. 基础代码与实战验证
```text
// 跳跃表节点
class SkipListNode {
  constructor(value, level) {
    this.value = value;
    this.forward = new Array(level).fill(null); // forward[i] 指向第i层的后继节点
  }
}

class SkipList {
  constructor(maxLevel = 16, p = 0.5) {
    this.maxLevel = maxLevel;
    this.p = p;
    this.level = 1; // 当前实际最高层数，初始为1
    this.header = new SkipListNode(null, this.maxLevel); // 哨兵头节点，拥有最大层数
  }

  // 随机生成节点层数：从1开始，以概率p继续升层
  randomLevel() {
    let lvl = 1;
    while (Math.random() < this.p && lvl < this.maxLevel) lvl++;
    return lvl;
  }

  insert(value) {
    const update = new Array(this.level); // 记录各层前驱
    let cur = this.header;

    // 从最高有效层逐层向下，寻找插入位置的前驱
    for (let i = this.level - 1; i >= 0; i--) {
      while (cur.forward[i] && cur.forward[i].value < value) {
        cur = cur.forward[i]; // 同层向右移动
      }
      update[i] = cur; // 第i层应插入在cur之后
    }

    const lvl = this.randomLevel();
    // 若新层数超过当前最高层，扩展有效层并填充前驱为header
    if (lvl > this.level) {
      for (let i = this.level; i < lvl; i++) update[i] = this.header;
      this.level = lvl;
    }

    const newNode = new SkipListNode(value, lvl);
    // 逐层进行链表插入操作
    for (let i = 0; i < lvl; i++) {
      newNode.forward[i] = update[i].forward[i];
      update[i].forward[i] = newNode;
    }
  }

  search(target) {
    let cur = this.header;
    // 从最高层出发，每层尽可能向右移动，最终降到第0层
    for (let i = this.level - 1; i >= 0; i--) {
      while (cur.forward[i] && cur.forward[i].value < target) {
        cur = cur.forward[i];
      }
    }
    cur = cur.forward[0]; // 第0层的下一个节点即为可能的目标
    return cur && cur.value === target ? cur.value : null;
  }
}
```

### 4. 常见误区与进阶思考
误区1：认为随机化意味着性能不可预测。实际上，期望复杂度 O(log n) 是精确的数学保证，且由于每次操作独立，大量操作的平均性能趋于稳定；相反，固定层数方案（如按索引计数确定层数）需要额外维护，且最坏情况更差。
误区2：认为层数越高代表节点越重要。层数只是随机化的结果，与节点值大小无关；它影响的是搜索路径的期望长度，而不是数据的正确性。
深度思考题：如果 p 从 0.5 调整为 0.25，跳跃表的期望查找路径长度（比较次数）和空间开销（每节点平均指针数）分别如何变化？请推导出它们与 p 的关系，并解释为什么 Redis 选择 p=1/4 而不是 1/2 作为默认参数，从缓存层次和内存占用的角度分析。
