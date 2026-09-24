---
title: "每日基础技术总结 · 2026-09-25 · 堆：Top-K 问题与中位数维护"
date: 2026-09-25 07:19:37
categories: [技术分享]
tags: ["技术分享", "算法与数据结构（面试）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-25 · 堆：Top-K 问题与中位数维护

## 📚 今日主题

> **堆：Top-K 问题与中位数维护**（算法与数据结构（面试））

### 1. 核心概念速览
### 核心概念速览

堆（Heap）是一棵完全二叉树，满足堆序性质：父节点优先级恒优于或等于子节点，通常以数组连续存储。其本质是**部分有序的动态集合**——只保证极值可 O(1) 获取，不维护全序。

它解决两类高频问题：

1. **Top-K 问题**：在数据流或海量静态数据中找出最大/最小的 K 个元素。机制是维护一个容量恰好为 K 的门限堆——求 Top-K 大用小根堆，堆顶是当前候选中的最小值；新元素若大于堆顶则替换，否则丢弃。最终堆内 K 个元素即答案，复杂度 O(n log K)。

2. **中位数维护**：在动态插入数据过程中，实时返回当前有序序列的中位数。机制是双堆法：左侧最大堆保存较小的一半，右侧最小堆保存较大的一半，维护二者大小差不超过 1；中位数恒为左堆堆顶，或左右堆顶的平均值。插入 O(log n)，查询 O(1)。

堆是优先队列的底层实现，在计算机体系中是任务调度、Dijkstra、哈夫曼编码、流式统计的基石。专业工程师必须掌握，因为它直接关乎复杂度的数量级思维，也是理解分布式 Top-K、滑动窗口统计、实时推荐等进阶主题的基础。

### 2. 底层原理剖析
### 底层原理剖析

#### 物理存储

堆用数组实现，且根在下标 0 处。对任意下标 `i`：

- 左子：`2*i + 1`
- 右子：`2*i + 2`
- 父节点：`(i - 1) >> 1`

完全二叉树的特性使数组可无孔洞紧凑存储，无需指针，仅通过下标计算即可完成树结构遍历。

#### siftUp / siftDown 操作

- `push(x)`：将新元素追加到数组尾部，然后执行 siftUp——若当前节点破坏堆序（以最小堆为例，即小于父节点），则与父节点交换，沿路径向上直到合法。
- `pop()`：记录堆顶，将数组末尾元素移到堆顶，再执行 siftDown——与左右子中优先级最高者比较，若破坏堆序则交换，沿路径向下直到合法。

两操作均沿树高移动，复杂度 O(log n)。

#### Top-K 正确性证明（求前 K 大）

使用大小为 K 的小根堆作为候选集。堆顶是候选集中最小的元素。遍历所有元素时：

- 若当前元素 `x <= heap.top()`，候选集中已有 K 个元素都不小于 `x`，因此 `x` 不可能进入前 K，丢弃。
- 若 `x > heap.top()`，则 `x` 一定优于候选集中最差者，故弹出堆顶并入 `x`。

不变式：**任意时刻堆中元素是已扫描元素中最大的 K 个**。遍历结束后答案即为堆内 K 个元素。

#### 双堆中位数不变量

设 `L` 为最大堆（保存较小一半），`R` 为最小堆（保存较大一半）。插入新元素 `x`：

- 若 `L` 为空或 `x <= L.peek()`，则 `x` 属于较小半区，入 `L`；否则入 `R`。
- 随后平衡：
  - 若 `L.size > R.size + 1`：从 `L` 弹出堆顶送到 `R`。
  - 若 `R.size > L.size`：从 `R` 弹出堆顶送到 `L`。

平衡后恒有 `L.size == R.size` 或 `L.size == R.size + 1`。中位数计算：

- 元素总数为奇数：`L.peek()`。
- 元素总数为偶数：`(L.peek() + R.peek()) / 2`。

#### 与前端已有概念的异同

前端工程师熟知的 `Array.prototype.sort` 产生全序，而堆只维护父子间的弱序，刻意放弃兄弟间的全序。这让堆在动态插入/删除极值时从 O(n) 跃迁到 O(log n)，是“牺牲全局顺序换取单点优先级效率”的典型。

这与浏览器事件循环中的任务队列形成对比：FIFO 队列保证先到先服务，堆则允许按任意比较器定义的优先级出队，是更通用的调度模型。TypeScript 的类型系统是编译期约束，堆是运行期动态约束，二者本质都在消除部分信息以换取性能或安全，但堆的约束可在每次操作后重新恢复。

### 3. 基础代码与实战验证
```text
class Heap {
  constructor(cmp) {
    this.heap = [];
    this.cmp = cmp;  // 比较器：返回负值表示 a 应排在 b 之前
  }
  size() {
    return this.heap.length;
  }
  peek() {
    return this.heap[0]; // O(1) 访问堆顶
  }
  push(v) {
    const h = this.heap;
    h.push(v);
    let i = h.length - 1;
    // siftUp：与父节点比较，破坏堆序则交换
    while (i > 0) {
      const p = (i - 1) >> 1;
      if (this.cmp(h[p], h[i]) <= 0) break;
      [h[p], h[i]] = [h[i], h[p]];
      i = p;
    }
  }
  pop() {
    const h = this.heap;
    const top = h[0];
    const last = h.pop(); // 取出数组末尾元素
    if (h.length > 0) {
      h[0] = last;        // 移到根位置，准备下沉
      let i = 0;
      // siftDown：与左右子中最优先者交换
      while (true) {
        const l = i * 2 + 1;
        const r = i * 2 + 2;
        let best = i;
        if (l < h.length && this.cmp(h[l], h[best]) < 0) best = l;
        if (r < h.length && this.cmp(h[r], h[best]) < 0) best = r;
        if (best === i) break;
        [h[i], h[best]] = [h[best], h[i]];
        i = best;
      }
    }
    return top;
  }
}

// Top-K：求数组中最大的 K 个元素
function topK(array, K) {
  const minHeap = new Heap((a, b) => a - b); // 小根堆
  for (const num of array) {
    if (minHeap.size() < K) {
      minHeap.push(num);   // 堆未满，直接加入候选集
    } else if (num > minHeap.peek()) { // 比当前最差候选大才替换
      minHeap.pop();
      minHeap.push(num);
    }
  }
  return minHeap.heap; // 内部数组虽不保证有序，但元素集合正确
}

// 中位数维护：双堆法
class MedianFinder {
  constructor() {
    this.left = new Heap((a, b) => b - a);  // 最大堆，保存较小的一半
    this.right = new Heap((a, b) => a - b); // 最小堆，保存较大的一半
  }
  addNum(num) {
    if (this.left.size() === 0 || num <= this.left.peek()) {
      this.left.push(num);
    } else {
      this.right.push(num);
    }
    // 保持不变量：left.size == right.size 或 left.size == right.size + 1
    if (this.left.size() > this.right.size() + 1) {
      this.right.push(this.left.pop());
    } else if (this.right.size() > this.left.size()) {
      this.left.push(this.right.pop());
    }
  }
  findMedian() {
    if (this.left.size() === this.right.size()) {
      return (this.left.peek() + this.right.peek()) / 2;
    }
    return this.left.peek();
  }
}
```

### 4. 常见误区与进阶思考
### 常见误区与进阶思考

**误区 1：把堆当成完全有序的数组。** 堆只约束父与子之间的关系，兄弟之间没有既定顺序。因此底层数组层序遍历的结果不是有序序列，不能直接用于二分查找或线性输出排序结果。若需要有序的 Top-K，必须额外对堆内 K 个元素排序，这一操作是 O(K log K)，但不影响主流程复杂度。

**误区 2：Top-K 中选错堆的方向。** 求前 K 大必须用小根堆，求前 K 小必须用大根堆。堆顶是候选集中“最不应该被保留”的那个元素，它作为门限被新元素挑战。选反方向会让堆保留的是“最该被淘汰”的元素，结果完全错误。

**进阶思考题：** 在滑动窗口中维护 Top-K，窗口右移时旧元素需要被移出候选集。但堆不支持直接删除任意元素（不改变其他元素相对顺序的情况下，需全扫描）。请设计一种基于“懒惰删除 + 两个堆”的扩展，说明如何用额外标记使已删除元素延迟到堆顶时再被真正弹出，并分析若窗口滑动频繁，堆顶长期被“死元素”占据时真实 Top-K 如何保证。
