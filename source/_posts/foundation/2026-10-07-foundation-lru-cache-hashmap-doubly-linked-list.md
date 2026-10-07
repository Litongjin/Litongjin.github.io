---
title: "每日基础技术总结 · 2026-10-07 · LRU 缓存：哈希表 + 双向链表的 O"
date: 2026-10-07 07:02:45
categories: [技术分享]
tags: ["技术分享", "算法与数据结构（面试）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-07 · LRU 缓存：哈希表 + 双向链表的 O

## 📚 今日主题

> **LRU 缓存：哈希表 + 双向链表的 O**（算法与数据结构（面试））

### 1. 核心概念速览
LRU（Least Recently Used）缓存是一种淘汰策略，核心约束：容量固定，命中率优先，访问即更新新鲜度，淘汰时选择最久未被访问的项。其工程本质是在『O(1) 查找』与『O(1) 顺序维护』之间做数据结构组合：哈希表提供键到节点的直接寻址，双向链表维护访问时序的相对顺序。二者结合后，get/put 均可摊还 O(1)，且不需要扫描或移动数组元素。
在计算机体系中，LRU 不只是一种算法题，而是缓存系统的通用底层模型：CPU TLB、页表替换、HTTP 缓存、数据库 buffer pool、CDN 边缘节点、AI 推理 KV Cache 淘汰，都会使用 LRU 或其近似变体。专业工程师必须掌握它，因为它直接暴露了缓存的三个本质问题：索引结构、时序结构、并发一致性。

### 2. 底层原理剖析
运行机制可以拆成两个正交能力：
1. 快速定位：哈希表 key -> node，解决『存在性与寻址』。
2. 顺序维护：双向链表按最近访问时间排序，头部表示最新，尾部表示最旧，解决『淘汰对象选择』。

标准操作语义：
- get(key)：若不存在返回未命中；若存在，将该节点从当前位置摘除并插入头部，然后返回值。
- put(key, value)：若 key 存在，更新值并移动到头；若不存在，新建节点插入头部；若容量超限，删除链表尾节点，并同步删除哈希表中的键。

关键点在于：双向链表的每个节点必须同时持有 prev/next 指针和 key。持有 key 是必要的，因为淘汰尾节点时需要反向删除哈希表项；否则会出现链表与哈希表状态不一致。

伪代码：
get(key):
  node = map.get(key)
  if node == null: return miss
  remove(node)
  pushHead(node)
  return node.value

put(key, value):
  node = map.get(key)
  if node != null:
    node.value = value
    remove(node)
    pushHead(node)
    return
  node = new Node(key, value)
  map.set(key, node)
  pushHead(node)
  if size > capacity:
    tail = popTail()
    map.delete(tail.key)
    size--

与前端已有知识对比：JavaScript 的 Map 保持插入顺序，可用 map.delete(key) + map.set(key, value) 模拟 LRU 的『访问后移到最新位置』。但这是语言运行时内置的有序哈希实现，底层通常依赖更复杂的结构；手写哈希表 + 双向链表的价值在于把『有序性』和『淘汰语义』显式化，理解为什么有序哈希可以支撑 LRU，以及在需要精细控制内存、并发、近似淘汰时如何替换底层结构。

### 3. 基础代码与实战验证
```text
class Node {
  constructor(key, value) {
    this.key = key;       // 保存 key：尾节点被淘汰时用于反向删除 map 项
    this.value = value;
    this.prev = null;     // 双向链表前驱
    this.next = null;     // 双向链表后继
  }
}

class LRUCache {
  constructor(capacity) {
    this.capacity = capacity;
    this.map = new Map(); // key -> Node，O(1) 定位
    this.head = new Node(0, 0); // 哨兵头，避免空链表边界判断
    this.tail = new Node(0, 0); // 哨兵尾，避免空链表边界判断
    this.head.next = this.tail;
    this.tail.prev = this.head;
  }

  _remove(node) {
    // O(1) 摘除：双向指针直接重连，不需要移动其他节点
    node.prev.next = node.next;
    node.next.prev = node.prev;
  }

  _addToHead(node) {
    // 插入到 head 之后，表示最近访问节点位于逻辑头部
    node.prev = this.head;
    node.next = this.head.next;
    this.head.next.prev = node;
    this.head.next = node;
  }

  get(key) {
    const node = this.map.get(key);
    if (!node) return -1; // 未命中：哈希表不存在该 key
    this._remove(node);   // 先从旧时间位置摘除
    this._addToHead(node); // 再放入头部，完成新鲜度更新
    return node.value;
  }

  put(key, value) {
    const node = this.map.get(key);
    if (node) {
      node.value = value; // 更新已有节点值
      this._remove(node); // 访问语义：移动到头，而不是简单覆盖
      this._addToHead(node);
      return;
    }
    const newNode = new Node(key, value);
    this.map.set(key, newNode); // 哈希表建立索引，保证后续 get O(1)
    this._addToHead(newNode);   // 新节点天然为最近访问，插入头部
    if (this.map.size > this.capacity) {
      const lru = this.tail.prev; // 尾部前一个节点即最久未访问节点
      this._remove(lru);
      this.map.delete(lru.key);   // 必须通过 node.key 删除哈希项，否则会泄漏脏索引
    }
  }
}
```

### 4. 常见误区与进阶思考
误区一：把 LRU 简单理解为『队列 + 哈希表』。普通队列只能按入队顺序出队，不能在访问已有元素时把它移动到队首；若强行删除再入队，没有双向链表就无法 O(1) 定位并摘除中间节点，最终退化到 O(n)。

误区二：忽略并发与内存语义。单线程实现中的 O(1) 不等于生产可用；多线程下 remove/addHead/map 操作必须原子化，否则会出现悬空指针、重复删除、map 与链表不一致。缓存系统还要考虑脏数据回写、TTL、淘汰回调，这些会把纯数据结构问题升级为一致性问题。

进阶思考：为什么 Redis、操作系统页替换和 CDN 常使用『近似 LRU』而不是精确 LRU？如果要求严格精确，双向链表的每次访问都会产生写操作和指针变更，在高并发或持久化场景下会带来锁争用、cache line invalidation、写放大问题；近似 LRU 用采样、时钟位或访问频率统计换取更低的维护成本。这背后是『算法正确性』与『系统吞吐/一致性成本』的权衡。
