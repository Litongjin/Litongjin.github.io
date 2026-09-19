---
title: "每日基础技术总结 · 2026-09-19 · 跳跃表的层数随机化与查找/插入路径"
date: 2026-09-19 08:00:00
categories: [技术分享]
tags: ["技术分享", "算法与数据结构"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-19 · 跳跃表的层数随机化与查找/插入路径

## 📚 今日主题

> **跳跃表的层数随机化与查找/插入路径**（算法与数据结构）

### 1. 核心概念速览
定义：跳跃表（Skip List）是一种概率型有序数据结构，由多层有序链表构成。每个节点拥有一个随机高度 h，其 forward[0..h-1] 分别指向同层后继。层数随机化独立于键：通常 P(h >= k) = p^{k-1}（k>=1），期望高度 1/(1-p)。它解决的问题：在动态有序集合上支持查找、插入、删除、范围查询，期望时间复杂度 O(log n)，且实现比红黑树/AVL 简单。本质：用随机化的多级稀疏索引替代确定性平衡旋转，使每层节点数按概率 p 递减，从而把查找路径压缩到 O((1/p) log_{1/p} n)。在计算机体系中的位置：介于链表与平衡树之间，是 Redis zset、LevelDB/RocksDB memtable、部分并发数据结构的基础；在 AI 体系中，有序索引和概率数据结构用于检索、排序、Top-K 等。专业工程师必须掌握：它训练概率平衡、期望分析、指针重连、并发友好设计，是理解存储引擎和数据库索引的底层组件。

### 2. 底层原理剖析
1. 节点结构：head 节点持有 maxLevel 个 forward 指针；普通节点高度 h 意味着它出现在第 0 到 h-1 层。
2. 随机层数：randomLevel() 独立采样，与 key 无关。伪代码：lvl = 1; while random() < p and lvl < maxLevel: lvl++; return lvl。P(h >= k) = p^{k-1}，期望高度 1/(1-p)。每层节点数期望 n * p^{i-1}，总层数期望 log_{1/p} n。
3. 查找：从当前最高层 this.level-1 开始，对每层 i 从高到低：while x.forward[i] != null and x.forward[i].key < key: x = x.forward[i]；然后下降一层。到第 0 层后，目标为 x.forward[0]，比较 key 是否相等。路径是“右移直到下一个节点不小于 key，再下降”的阶梯。
4. 插入：先执行查找，同时用 update[i] 记录每一层最后一个小于 key 的节点（前驱）。随机生成 newLevel。若 newLevel > this.level，则对 i 从 this.level 到 newLevel-1 设置 update[i] = head，并更新 this.level = newLevel。然后创建节点，对 i 从 0 到 newLevel-1：node.forward[i] = update[i].forward[i]; update[i].forward[i] = node。这是逐层链表插入。
5. 删除：同样记录 update[i]，逐层断开，若最高层变为空则降低 this.level。
6. 期望分析：查找路径长度 = 每层向前步数之和 + 下降层数。每层节点密度为 p，从任意节点出发，期望前进 1/p 步遇到下一个提升节点；层数约 log_{1/p} n，故期望 O((1/p) log_{1/p} n)。p 越大，层数越多，空间换时间；p=1/2 时总指针期望约 2n，p=1/4 时空间更省但查找步数增加。
7. 与平衡树对比：红黑树/AVL 通过旋转维护最坏 O(log n)，实现复杂；跳跃表通过随机化维护概率平衡，最坏 O(n) 但概率极低，插入/删除只需局部指针修改，更适合并发。
8. 与前端已有概念异同：前端中数组随机访问 O(1) 但插入 O(n)；链表插入 O(1) 但查找 O(n)。跳跃表相当于在链表上叠加动态维护的多层稀疏索引，但索引层由节点随机参与，非离线固定。它不同于 JavaScript Map 的哈希索引：Map 平均 O(1) 查找但不支持有序范围查询；跳跃表保持有序，支持范围扫描。与 TS/Java 接口无直接关系；若类比接口，接口是抽象契约，跳跃表是具体概率数据结构。

### 3. 基础代码与实战验证
```text
class SkipListNode {
  constructor(key, level) {
    this.key = key;
    this.forward = new Array(level).fill(null);
  }
}

class SkipList {
  constructor(maxLevel = 16, p = 0.5) {
    this.maxLevel = maxLevel;
    this.p = p;
    this.level = 1;
    this.head = new SkipListNode(null, maxLevel);
  }

  randomLevel() {
    let lvl = 1;
    while (Math.random() < this.p && lvl < this.maxLevel) {
      lvl++;
    }
    return lvl;
  }

  find(key) {
    let x = this.head;
    for (let i = this.level - 1; i >= 0; i--) {
      while (x.forward[i] && x.forward[i].key < key) {
        x = x.forward[i];
      }
    }
    const target = x.forward[0];
    return target && target.key === key ? target : null;
  }

  insert(key) {
    const update = new Array(this.maxLevel).fill(this.head);
    let x = this.head;
    for (let i = this.level - 1; i >= 0; i--) {
      while (x.forward[i] && x.forward[i].key < key) {
        x = x.forward[i];
      }
      update[i] = x;
    }
    const newLevel = this.randomLevel();
    if (newLevel > this.level) {
      for (let i = this.level; i < newLevel; i++) {
        update[i] = this.head;
      }
      this.level = newLevel;
    }
    const node = new SkipListNode(key, newLevel);
    for (let i = 0; i < newLevel; i++) {
      node.forward[i] = update[i].forward[i];
      update[i].forward[i] = node;
    }
  }

  dump() {
    for (let i = this.level - 1; i >= 0; i--) {
      const arr = [];
      let x = this.head.forward[i];
      while (x) {
        arr.push(x.key);
        x = x.forward[i];
      }
      console.log('L' + i + ': ' + arr.join(' -> '));
    }
  }
}

const sl = new SkipList();
[3, 6, 7, 9, 12, 19, 17, 26, 21, 25].forEach(k => sl.insert(k));
sl.dump();
console.log(sl.find(19)?.key);
console.log(sl.find(20));

// 关键行解释：
// randomLevel: while (Math.random() < p) 以 p 概率继续提升，层数独立同分布，P(h>=k)=p^{k-1}。
// find: 从 this.level-1 开始，x.forward[i].key < key 则右移，否则下降，形成阶梯路径。
// insert: update[i] 记录每层最后一个小于 key 的节点；newLevel 高于当前 level 时，新层前驱只能是 head。
// 逐层 node.forward[i] = update[i].forward[i]; update[i].forward[i] = node; 是标准链表插入。
```

### 4. 常见误区与进阶思考
常见误区 1：认为跳跃表的随机化层数能保证最坏 O(log n)。实际上它只保证期望 O(log n)，最坏 O(n)；若随机源可预测或被攻击者控制，攻击者可让所有节点高度为 1，退化为单链表。防御手段包括使用密码学安全随机源、以键的哈希作为随机种子、设置 maxLevel、定期重建等，但无法完全消除概率性最坏。
常见误区 2：插入时只更新第 0 层，或新节点高度大于当前 level 时忘记扩展 head 的 forward 并将 update[i] 设为 head。这会导致高层指针丢失或查找错误。另一个相关误区：认为每层节点数严格为 n/2^i；实际是期望值，存在方差，p 的选择需权衡空间与查找步数。
进阶思考题：在无锁并发跳跃表中，插入需要原子地更新多个 forward 指针，查找可能观察到部分更新。跳跃表为什么通常比平衡树更适合并发？请从“局部指针修改 vs 全局旋转”以及“是否允许查找看到中间状态”两个角度分析，并给出一种可行的并发控制协议（如 CAS 自旋、标记指针、RCU）。
