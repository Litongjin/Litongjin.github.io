---
title: "每日基础技术总结 · 2026-11-02 · Redis ZSet跳跃列表SkipList的插入概率控制与内存优化"
date: 2026-11-02 07:01:51
categories: [技术分享]
tags: ["技术分享", "后端基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-02 · Redis ZSet跳跃列表SkipList的插入概率控制与内存优化

## 📚 今日主题

> **Redis ZSet跳跃列表SkipList的插入概率控制与内存优化**（后端基础）

### 1. 核心概念速览
Redis ZSet 同时使用 dict 与 skiplist：dict 提供 member→score 的 O(1) 点查，skiplist 按 (score, member) 有序组织并支持范围查询与排名。SkipList 的本质是在有序链表上叠加多层前向指针的概率型索引；插入节点时独立随机决定层高，层高分布为 P(L=k)=(1-p)p^{k-1}（1≤k≤maxLevel），Redis 取 p=1/4、maxLevel=32，因此期望层高为 1/(1-p)=1.333，节点期望 forward/span 指针数受控，查询/插入/删除为期望 O(log N)，最坏 O(N)。它解决的问题是：在动态有序集合中同时获得近似平衡树的 O(log N) 操作与链表式局部改写，避免确定性平衡结构的旋转/分裂成本。内存优化不是单一参数：小 ZSet 使用 listpack 连续编码，超过 zset-max-listpack-entries/value 阈值转 skiplist+dict；skiplist 节点含 backward 与每层 forward/span，span 支撑排名但增加内存；p 控制层高分布，在查询期望与指针开销间折中。该结构位于 Redis 有序集合、排行榜、延迟队列、范围检索的核心，是概率数据结构与内存-时间权衡的典型，专业工程师必须掌握其期望复杂度、最坏情况和编码转换边界。

### 2. 底层原理剖析
底层结构：zset 对象持 dict 与 zskiplist；zskiplist 含 header、tail、length、level。zskiplistNode 含 sds ele、double score、backward、level[]，每层为 {forward, span}。header 不存业务 member，层数组长度固定为 32 上限。随机层高：lvl=1；while random()<0.25 and lvl<32: lvl++。由此 P(L≥k)=p^{k-1}，E[L]=1/(1-p)=1.333；每节点期望 forward 指针数与 span 数均约 1.333，最大 32，避免每节点固定 32 层的确定性膨胀。插入流程：1 从 header 当前最高层向下，沿 forward 比较 (score, ele)，记录每层前驱 update[i] 与累计排名 rank[i]；2 生成随机层高 lvl；3 若 lvl>当前 level，则对新增层令 update[i]=header、rank[i]=0，并提升 level；4 创建节点，对 i=0..lvl-1 插入：node.forward[i]=update[i].forward[i]；update[i].forward[i]=node；node.span[i]=update[i].span[i]-(rank[0]-rank[i])；update[i].span[i]=(rank[0]-rank[i])+1；5 对未覆盖层 update[i].span[i]++；6 维护 backward 与 tail。查询从最高层开始：若下一节点 (score,ele) 小于目标则前进，否则下降；范围查询先 O(log N) 定位起点，再沿底层 forward 扫描 M 个，复杂度 O(log N+M)；排名通过 span 累加。内存优化：listpack 编码按 score/member 连续存储，小集合无指针开销且缓存友好，但操作 O(N)；超过阈值转 skiplist，且转换不可逆。与前端对比：JS Map 提供 O(1) 点查但不按 score 排序；用 Map+排序数组可实现点查与范围，但插入需 splice O(N) 并移动元素；跳表相当于把有序数组的二分索引改成链表上的多层稀疏索引，插入只改局部指针，代价是额外 forward/span 指针与随机化最坏 O(N)。ZSet 的 dict+skiplist 双结构等价于 Map 点查与跳表范围索引的组合，二者共享 member/score 语义但各自承担不同访问模式。

### 3. 基础代码与实战验证
```text
import random

class Node:
    __slots__ = ('score', 'member', 'forward')
    def __init__(self, score, member, level):
        self.score = score
        self.member = member
        self.forward = [None] * level  # 每层一个 forward；生产实现还含 span/backward

class SkipList:
    P = 0.25
    MAX_LEVEL = 32
    def __init__(self):
        self.header = Node(float('-inf'), '', self.MAX_LEVEL)
        self.level = 1
    def random_level(self):
        lvl = 1
        while lvl < self.MAX_LEVEL and random.random() < self.P:
            lvl += 1  # 几何分布：P(L=k)=(1-p)p^(k-1)
        return lvl
    def insert(self, score, member):
        update = [None] * self.MAX_LEVEL
        x = self.header
        for i in range(self.level - 1, -1, -1):
            while x.forward[i] and (x.forward[i].score, x.forward[i].member) < (score, member):
                x = x.forward[i]  # 同层前进，保持 (score,member) 有序
            update[i] = x  # 记录第 i 层插入位置前驱
        lvl = self.random_level()
        if lvl > self.level:
            for i in range(self.level, lvl):
                update[i] = self.header  # 新层的前驱只能是 header
            self.level = lvl
        node = Node(score, member, lvl)
        for i in range(lvl):
            node.forward[i] = update[i].forward[i]
            update[i].forward[i] = node  # 只改局部指针，不移动其他节点
    def pointer_count(self):
        cnt = 0
        x = self.header.forward[0]
        while x:
            cnt += len(x.forward)  # 统计每节点实际 forward 指针数
            x = x.forward[0]
        return cnt

if __name__ == '__main__':
    random.seed(1)
    sl = SkipList()
    N = 10000
    for i in range(N):
        sl.insert(random.random(), str(i))
    print(sl.level, sl.pointer_count() / N)  # 平均约 1/(1-0.25)=1.333
```

### 4. 常见误区与进阶思考
误区一：把 p 当成可调内存旋钮。Redis 的 ZSKIPLIST_P 固定为 1/4，且 skiplist 编码只在超过 listpack 阈值后启用；盲目调小 p 会降低层数、减少指针但增加底层扫描比较，最坏退化 O(N)，且不解决 dict 与节点元数据开销。误区二：认为 ZSet 始终是 skiplist。小集合由 listpack 存储，点查/范围小数据为 O(N) 但无指针开销；阈值转换不可逆，删除元素也不会自动退回 listpack。另一个易混点：跳表查询是期望 O(log N)，不是严格最坏 O(log N)，随机层高导致最坏 O(N)。思考题：若把 p 从 1/4 降到 1/8，期望层高与每节点期望 forward/span 指针数如何变化？范围查询扫描 M 个元素时，定位起点的比较次数与底层扫描成本如何权衡？为什么 Redis 选择固定 p 而不暴露为配置项？
