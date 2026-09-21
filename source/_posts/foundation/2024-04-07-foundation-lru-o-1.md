---
title: "每日基础技术总结 · 2024-04-07 · LRU 缓存：哈希表 + 双向链表的 O(1)"
date: 2024-04-07 20:00:00
categories: [技术分享]
tags: ["技术分享", "算法与数据结构（面试）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-04-07 · LRU 缓存：哈希表 + 双向链表的 O(1)

## 📚 今日主题

> **LRU 缓存：哈希表 + 双向链表的 O(1)**（算法与数据结构（面试））

### 1. 核心概念速览
LRU（Least Recently Used）缓存是一种基于访问频率的时间局部性原理设计的淘汰策略，其核心在于维护一个容量上限的有序集合，确保最近被访问的元素保留在高位，最久未访问的元素置于低位以便快速淘汰。该结构通过哈希表（Hash Map）与双向链表（Doubly Linked List）的组合，实现了插入、查找和删除操作的时间复杂度严格为 O(1)。在计算机体系中，它是解决内存容量受限场景下数据复用效率问题的经典数据结构基础；对于全栈工程师而言，掌握此机制是理解 Redis 等中间件缓存底层实现、设计高性能本地缓存以及优化系统资源利用率的前提条件。

2. **Principles**: LRU 的核心机制依赖于两个关键特性：哈希表的 O(1) 随机访问能力，与双向链表的 O(1) 节点移动能力。当执行 `get(key)` 或 `put(key, value)` 时：
1. 若 key 存在于哈希表中，直接定位到对应的双向链表节点。
2. 将该节点从当前链表位置移除（unlink），并重新插入到链表头部（insert head），表示该元素变为“最新使用”。这一过程仅涉及指针重定向，无需遍历。
3. 若 key 不存在且调用 `put`，则在哈希表中新建映射并将新节点插入链表头部。
4. 若链表长度超过预设容量 threshold，触发淘汰机制：移除链表尾部节点（即最久未使用），并从哈希表中删除对应键值对。

对比前端/TypeScript 概念体系：这与 JS 引擎中 V8 的 HashMap 存储对象属性类似，但增加了显式的顺序管理约束。不同于 TS 接口仅做类型契约检查（编译期），LRU 的结构是在运行时动态维持状态一致性，任何变更都需同步更新两种异构数据结构的引用关系，类似于 React 虚拟 DOM diff 算法中对 DOM 节点引用的精确追踪与重排，但操作粒度更细至单个节点指针层面。

3. **Code**: ```python
# Python 实现演示本质逻辑

class Node:
    def __init__(self, key=None, val=None):
        self.key = key
        self.val = val
        self.prev = None
        self.next = None

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = {} # Hash Table: Key -> Node reference
        # Sentinel nodes to avoid boundary checks
        self.head = Node() # Dummy head (most recently used end)
        self.tail = Node() # Dummy tail (least recently used end)
        self.head.next = self.tail
        self.tail.prev = self.head

    def get(self, key: int) -> int:
        if key not in self.cache:
            return -1
        node = self.cache[key]
        self._remove(node)       # Remove from current position
        self._add_to_head(node)  # Move to head (latest access)
        return node.val

    def put(self, key: int, value: int) -> None:
        if key in self.cache:
            node = self.cache[key]
            node.val = value
            self._remove(node)
            self._add_to_head(node)
        else:
            node = Node(key, value)
            self.cache[key] = node
            self._add_to_head(node)
            if len(self.cache) > self.capacity:
                lru_node = self.tail.prev
                self._remove(lru_node)
                del self.cache[lru_node.key]

    def _add_to_head(self, node):
        # Insert after head
        node.prev = self.head
        node.next = self.head.next
        self.head.next.prev = node
        self.head.next = node

    def _remove(self, node):
        # Unlink node from list
        prev_node = node.prev
        next_node = node.next
        prev_node.next = next_node
        next_node.prev = prev_node
```

4. **Pitfalls**:
常见误区一：混淆哈希表的作用。初学者常误以为需要遍历链表寻找 key，实际上哈希表的存在正是为了消除 O(N) 查找成本，若无哈希表辅助，LRU 退化为 O(N) 复杂度的普通双向链表。
常见误区二：忽略哨兵节点（Sentinel/Dummy Nodes）的价值。手动处理头尾边界情况极易导致空指针异常或死循环，引入哑节点可将所有节点视为内部节点处理，简化指针操作逻辑。

深度思考题：在并发环境下（如多线程同时调用 `get` 和 `put`），上述纯手写实现的 LRU 存在竞态条件风险。若要求在不使用全局锁（Global Lock）的情况下保证线程安全，如何结合读写锁（Read-Write Lock）或无锁数据结构（Lock-free Data Structures）进行优化？请简述其权衡利弊。
