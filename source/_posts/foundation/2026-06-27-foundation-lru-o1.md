---
title: "每日基础技术总结 · 2026-06-27 · LRU 缓存的哈希表+双向链表与 O(1) 操作"
date: 2026-06-27 08:00:00
categories: [技术分享]
tags: ["技术分享", "算法与数据结构"]
author: Litongjin
---

# 每日基础技术总结 · 2026-06-27 · LRU 缓存的哈希表+双向链表与 O(1) 操作

## 📚 今日主题

> **LRU 缓存的哈希表+双向链表与 O(1) 操作**（算法与数据结构）

### 1. 核心概念速览
LRU（Least Recently Used）缓存是一种容量受限的键值存储，其淘汰策略为：当缓存满时，优先移除最久未被访问的条目。本质是维护一个访问时间序，并保证查找、插入、删除、淘汰四个操作均为 O(1) 平均时间复杂度。其机制是哈希表提供 O(1) 的键定位，双向链表提供 O(1) 的节点摘除与尾部追加/头部移除（取决于实现），二者通过节点对象内的指针相互引用，形成索引与顺序结构的复合体。LRU 位于操作系统页面置换、数据库缓冲池、CDN 边缘缓存、AI 推理中的特征缓存等场景的算法层，是并发控制与存储引擎设计的基石之一。专业工程师必须掌握，因为它是系统性能调优中“缓存一致性”与“时间局部性”两大原则的直接工程化实现，也是面试和架构设计中评估候选人底层功底的标准试金石。

### 2. 底层原理剖析
底层由两个数据结构协同：1) 哈希表（如 Java HashMap / Python dict / JS Map）以 key 映射到链表节点指针；2) 双向链表，每个节点存储 key、value 以及 prev/next 指针。链表头部表示最近使用，尾部表示最久未使用。核心操作：- 读取 get(key)：哈希表查找到节点，将该节点从当前位置摘除并移到链表头部，返回 value。摘除和头插均为 O(1)。- 写入 put(key, value)：若 key 已存在，更新节点值并移到头部；若不存在且容量未满，创建节点并插入头部；若已满，先删除尾部节点（同时从哈希表删除其 key），再插入新节点到头部。所有操作都是常数次指针调整。本质是“用哈希表加速定位，用双向链表加速顺序调整”。之所以用双向链表而非单向，是因为删除任意已知节点时需要同时获取其前驱和后继，单向链表无法 O(1) 获取前驱（除非额外维护）。对比前端知识：类似 React 的 Fiber 树中使用链表保存 work-in-progress 顺序，但 LRU 的链表节点同时被哈希表索引，相当于在 Fiber 上再挂一个 Map 以 key 直接定位节点；也类似于浏览器 HTTP 缓存（Cache-Control 中的 LRU 启发式）但这里是从算法层面强制精确控制。与 TypeScript 接口 vs Java 接口的差异类似：前者是结构类型（鸭子类型），后者是名义类型（必须显式 implements），而 LRU 的哈希表和链表在多种语言中实现方式不同但接口语义一致，强调“逻辑结构”与“语言实现”的解耦。

### 3. 基础代码与实战验证
```text
以下为 Python 极简实现，未依赖任何框架，直接展示哈希表 + 双向链表的协作。

class _Node:
    def __init__(self, key, val):
        self.key = key
        self.val = val
        self.prev = None
        self.next = None

class LRUCache:
    def __init__(self, capacity: int):
        self.cap = capacity
        self.hash = {}  # key -> _Node，哈希表负责 O(1) 定位节点
        self.head = _Node(None, None)  # 哨兵头，next 指向最近使用
        self.tail = _Node(None, None)  # 哨兵尾，prev 指向最久使用
        self.head.next = self.tail
        self.tail.prev = self.head

    def _remove(self, node):
        # 摘除节点：只改两个指针，O(1)
        prev_node = node.prev
        next_node = node.next
        prev_node.next = next_node
        next_node.prev = prev_node

    def _insert_head(self, node):
        # 插入到头部：让 node 成为最近使用，O(1)
        old_first = self.head.next
        self.head.next = node
        node.prev = self.head
        node.next = old_first
        old_first.prev = node

    def get(self, key: int) -> int:
        if key not in self.hash:
            return -1
        node = self.hash[key]
        self._remove(node)
        self._insert_head(node)
        return node.val

    def put(self, key: int, value: int) -> None:
        if key in self.hash:
            node = self.hash[key]
            node.val = value  # 更新值
            self._remove(node)
            self._insert_head(node)
            return
        if len(self.hash) >= self.cap:
            # 满则淘汰尾部节点：其 key 必须从哈希表移除
            lru_node = self.tail.prev
            self._remove(lru_node)
            del self.hash[lru_node.key]
        new_node = _Node(key, value)
        self.hash[key] = new_node
        self._insert_head(new_node)
```

### 4. 常见误区与进阶思考
1. 误区一：认为用数组或 JS Map 的插入顺序就能实现 O(1) 的 LRU。数组移动元素是 O(n)，Map 虽然保持插入顺序但删除任意中间项后重新插入是 O(1)，但 Map 本身不提供“移动到头部”的原生 O(1) 操作，需要先 delete 再 set，这虽然可行但在某些引擎中 delete 会导致哈希表退化或触发 rehash，且无法直接获取“最久未使用”的节点而不遍历。本质上 Map 不暴露链表指针，无法同时做到定位和调整顺序都 O(1)。
2. 误区二：双向链表中存储 value 但不在节点中存储 key。淘汰时若节点不携带 key，则无法从哈希表中删除对应条目，导致内存泄漏或需要额外遍历找 key。因此节点必须同时保存 key 和 value，这是实现中容易遗漏的细节。

思考题：在并发环境下，若哈希表使用读写锁、链表操作使用互斥锁，如何避免锁嵌套导致的死锁？请设计一种锁定顺序（如先锁链表再锁哈希表，或统一锁粒度），并说明为什么哈希表操作和链表操作必须保持原子性才能保证 LRU 语义的一致性。
