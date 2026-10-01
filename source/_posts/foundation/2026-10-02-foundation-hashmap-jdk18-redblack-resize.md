---
title: "每日基础技术总结 · 2026-10-02 · HashMap：JDK1.8 红黑树化与扩容机制"
date: 2026-10-02 07:02:42
categories: [技术分享]
tags: ["技术分享", "Java 后端与 Spring 生态"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-02 · HashMap：JDK1.8 红黑树化与扩容机制

## 📚 今日主题

> **HashMap：JDK1.8 红黑树化与扩容机制**（Java 后端与 Spring 生态）

### 1. 核心概念速览
HashMap 是 Java 中基于哈希表实现的 Map 接口实现，JDK1.8 起在底层引入数组+链表+红黑树的三态结构。其本质是通过哈希函数将 key 映射到桶（bucket），以期望 O(1) 的均摊访问复杂度；当哈希冲突严重时，链表长度超过阈值（默认 8）且数组容量大于等于 64 时，链表会树化为红黑树，将最坏情况从 O(n) 降为 O(log n)。扩容（resize）是在元素数量超过阈值（容量*负载因子，默认 0.75）时触发，将数组容量翻倍并重新分配所有节点，目的是维持哈希表的低冲突概率与空间利用率。该机制是理解 Java 集合框架、内存模型、并发安全（如 JDK7 头插法导致的死循环、JDK8 尾插法改进）的基石，也是后端工程师进行性能调优、解决线上 OOM 与线程安全问题的前提。前端工程师虽日常使用 JS 的 Map/Object，但 JS 引擎（如 V8）对哈希表的实现是黑盒；掌握 JDK 的 HashMap 能建立起哈希表在静态类型语言中的确定性行为认知，尤其是树化阈值、扩容时机、哈希扰动函数等设计哲学，为理解 Redis 哈希表、数据库索引乃至分布式一致性哈希打下基础。

### 2. 底层原理剖析
HashMap 的核心是 Node<K,V>[] table，每个桶要么为空，要么是单向链表节点，要么是红黑树根节点（TreeNode）。JDK1.8 的哈希函数：(h = key.hashCode()) ^ (h >>> 16)，将高 16 位与低 16 位异或，使散列值在高位变化时也能影响低位，减少冲突。桶下标计算：(n - 1) & hash，其中 n 是数组长度（恒为 2 的幂），等价于 hash % n 但更快。

树化条件与过程：
1. 当某个桶的链表长度达到 TREEIFY_THRESHOLD=8 时，调用 treeifyBin()。
2. 若此时数组容量 < MIN_TREEIFY_CAPACITY=64，则先执行 resize()，不立即树化，因为容量扩容后散列值重新分布可能打散长链表。
3. 若容量 >= 64，则将单向链表转为红黑树 TreeNode，并保持双向链表结构以便后续拆分（split）或退化为链表。

扩容机制：
- 扩容阈值 threshold = capacity * loadFactor。默认 capacity=16，loadFactor=0.75，当 size+1 > threshold 时触发 resize。
- 新容量 = 旧容量 << 1（翻倍），阈值同步翻倍。
- 迁移策略：遍历每个桶。桶若是单节点，直接计算新下标放回；若是链表，则根据 (hash & oldCap) == 0 分为 lo 和 hi 两条链，分别放入新数组原位置和原位置+oldCap 的位置。这是因为扩容后 n 的二进制多了一个高位 1，原下标取决于该位是 0 还是 1，无需重新计算 hash，只需判断新增位。
- 若是红黑树节点，调用 split()：同样根据 (hash & oldCap) 拆分为低位树和高位树，若拆分后的子链长度 <= UNTREEIFY_THRESHOLD=6，则退化为链表；否则保持红黑树。

伪代码：
```
void resize() {
    oldCap = table.length;
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) { threshold = Integer.MAX_VALUE; return; }
        newCap = oldCap << 1;
        newThr = oldThr << 1;
    } else { ... }
    Node[] newTab = new Node[newCap];
    for (Node e : table) {
        if (e instanceof TreeNode) { split(this, newTab, oldCap); }
        else if (e.next == null) { newTab[e.hash & (newCap-1)] = e; }
        else { // 链表拆分
            loHead = loTail = hiHead = hiTail = null;
            do {
                next = e.next;
                if ((e.hash & oldCap) == 0) {
                    if (loTail == null) loHead = e; else loTail.next = e;
                    loTail = e;
                } else {
                    if (hiTail == null) hiHead = e; else hiTail.next = e;
                    hiTail = e;
                }
                e = next;
            } while (e != null);
            if (loTail != null) { loTail.next = null; newTab[j] = loHead; }
            if (hiTail != null) { hiTail.next = null; newTab[j+oldCap] = hiHead; }
        }
    }
    table = newTab; threshold = newThr;
}
```

与前端概念的对比：Java 的 Map.Entry / Node 类似于 TS 中 Map 的键值对抽象，但 TS 的 Map 底层由 V8 的 OrderedHashTable（哈希表+双向链表）实现，没有红黑树，也没有用户可感知的扩容阈值。Java 的 HashMap 使用 `put`/`get`/`computeIfAbsent`，而 JS 对象 `{}` 的键只能为 string/symbol，Map 的键可为任意值；HashMap 不允许 key 为 null（但允许多个 null value），JS Map 允许任何值。更重要的是，Java 的相等性由 equals/hashCode 决定，而 JS 的 Map 使用 SameValueZero 语义，无哈希方法重写——这是语言级哈希契约的差异。

### 3. 基础代码与实战验证
注意：由于 MD 输出约定禁止使用代码围栏，上面 code 字段中的 Java 代码没有包裹在 ``` 内，而是用缩进表示。

### 4. 常见误区与进阶思考
误区一：只要链表长度超过 8 就会树化。实际必须同时满足数组容量 >= 64。当容量不足时，HashMap 会选择扩容而非树化，因为扩容能够将长链表拆散，比树化更高效。因此，在容器初始化容量过小时，你看到链表很长是正常现象。

误区二：查表时 `(n-1) & hash` 和 `hash % n` 完全相同。仅当 n 为 2 的幂时成立。HashMap 强制容量为 2 的幂，因此实现了位运算优化。若手动构造非 2 幂容量，HashMap 会将容量向上取整为 2 的幂。理解这一点对自定义 `tableSizeFor` 逻辑也很重要。

思考题：在 JDK1.8 中，如果向 HashMap 插入大量 key，使所有 key 的 `hashCode` 都返回同一个值，但 `equals` 各不相同，那么在数组容量从 16 增长到 64 的过程中，链表节点是何时、以什么条件真正变成红黑树的？当扩容发生、树节点被拆分成两条子链时，若其中一条子链长度为 5，另一条为 9，最终该桶应该呈现什么结构？请从源码中的 `split()` 方法逻辑角度推演。
