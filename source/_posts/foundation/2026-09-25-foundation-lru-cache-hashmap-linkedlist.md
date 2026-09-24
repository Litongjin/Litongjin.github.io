---
title: "每日基础技术总结 · 2026-09-25 · LRU 缓存的哈希表+双向链表与 O(1) 操作"
date: 2026-09-25 07:05:08
categories: [技术分享]
tags: ["技术分享", "算法与数据结构"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-25 · LRU 缓存的哈希表+双向链表与 O(1) 操作

## 📚 今日主题

> **LRU 缓存的哈希表+双向链表与 O(1) 操作**（算法与数据结构）

### 1. 核心概念速览
LRU（Least Recently Used）缓存是一种基于访问时间局部性的淘汰策略：当缓存容量达到上限时，优先淘汰最长时间未被访问的条目。其本质是通过哈希表提供 O(1) 的键查找，通过双向链表维护条目的访问顺序（最近访问的移到头部，最久未访问的在尾部），从而让 get 和 put 操作均达到 O(1) 时间复杂度。该结构是操作系统页面置换、数据库缓冲池、CDN 缓存、Redis 内存淘汰等场景的基石，也是面试和工程中高频考察的经典数据结构。专业工程师必须掌握它，因为它是理解『空间换时间』『复合数据结构协同』以及后续设计更复杂缓存策略（如 LFU、ARC）的基础。

### 2. 底层原理剖析
底层由两个数据结构协同：
- 哈希表（HashMap）：键 → 双向链表节点引用。它解决的是『快速定位节点』的问题，避免遍历链表查找。
- 双向链表（Doubly Linked List）：每个节点包含 key、value、prev、next。它解决的是『有序维护访问序列』的问题，使得在 O(1) 时间内将任意节点移动到头部或删除尾部。

核心操作：
1. get(key)：
   - 哈希表查找 key。若不存在，返回 -1。
   - 若存在，将对应节点从链表中摘除，并插入链表头部（表示最近使用）。
   - 返回节点值。
2. put(key, value)：
   - 若 key 已存在：更新节点值，并将节点移到头部。
   - 若 key 不存在：创建新节点并插入头部，同时加入哈希表。
   - 若此时容量超出限制：删除链表尾部节点（最久未使用），并从哈希表中移除对应 key。

实现细节：
- 使用虚拟头节点（dummy head）和虚拟尾节点（dummy tail）消除边界判断，使插入/删除操作统一化。
- 双向链表而非单链表的原因：删除任意节点需要 O(1) 获取前驱节点，单链表需要从头遍历或额外存储前驱。
- 哈希表的值存储节点引用（而非值），保证通过 key 直接定位节点，无需遍历。

对比前端已有概念：与浏览器 HTTP 缓存（如 Cache-Control 的 LRU 启发式）类似，但前端缓存通常在浏览器内核中以哈希表+双向链表实现；与 React Fiber 的链表结构不同——Fiber 是树形链表，LRU 是线性双向链表。本质上，组合两个基础数据结构实现一个新的抽象，正如 JavaScript 引擎用哈希表实现对象属性查找，再结合隐藏类优化，属于同一层级的底层思维。

### 3. 基础代码与实战验证
```text
class LRUCache {
  constructor(capacity) {
    this.capacity = capacity;
    this.hash = new Map(); // 键 → 节点
    // 虚拟头尾节点，消除边界判断
    this.head = { prev: null, next: null };
    this.tail = { prev: this.head, next: null };
    this.head.next = this.tail;
  }

  // 将节点从链表中摘除（借助 prev/next 指针，O(1)）
  removeNode(node) {
    node.prev.next = node.next;
    node.next.prev = node.prev;
  }

  // 将节点插入链表头部（虚拟头之后），标记为最近使用
  addToHead(node) {
    node.prev = this.head;
    node.next = this.head.next;
    this.head.next.prev = node;
    this.head.next = node;
  }

  // 访问节点：先摘除再插头，完成顺序更新
  moveToHead(node) {
    this.removeNode(node);
    this.addToHead(node);
  }

  get(key) {
    const node = this.hash.get(key);
    if (!node) return -1;
    this.moveToHead(node); // 访问即更新顺序
    return node.value;
  }

  put(key, value) {
    let node = this.hash.get(key);
    if (node) {
      node.value = value; // 更新值
      this.moveToHead(node); // 保持最近使用
      return;
    }
    // 新建节点，插入头部并加入哈希表
    node = { key, value, prev: null, next: null };
    this.hash.set(key, node);
    this.addToHead(node);

    // 超出容量：删除尾部节点（最久未使用）
    if (this.hash.size > this.capacity) {
      const lru = this.tail.prev; // 虚拟尾的前驱即真实节点
      this.removeNode(lru);
      this.hash.delete(lru.key); // 同步删除哈希表映射
    }
  }
}
// 备注：此处用纯 JS 对象模拟双向链表节点；生产环境可用 Map 的迭代顺序实现更简洁的 LRU，但本代码展示的是经典底层结构。
```

### 4. 常见误区与进阶思考
误区一：认为用 JavaScript 的 Map 天然实现 LRU 就是理解了底层。Map 确实按插入顺序迭代，通过 delete+set 可以模拟 O(1) 的 LRU，但那是引擎内置的红黑树或哈希表，非通用语义。若面试或移植到其他语言，必须掌握哈希表 + 双向链表的手工实现，否则无法理解为什么 O(1)。

误区二：只存储 value 而忽略 key 在链表节点中的作用。删除尾部节点时，必须通过该节点的 key 才能从哈希表中移除映射。若节点不存 key，则需要额外遍历或维护反向映射，破坏 O(1)。

思考题：如果哈希表的 value 存的是双向链表节点的『值』而不是『引用』，get 操作能否做到 O(1)？请推导为何必须存引用，以及当节点被移动时，哈希表如何处理才能保持一致性？
