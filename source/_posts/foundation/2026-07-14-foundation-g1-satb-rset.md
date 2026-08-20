---
title: "每日基础技术总结 · 2026-07-14 · G1 的 SATB 写屏障与 RSet"
date: 2026-07-14 08:00:00
categories: [技术分享]
tags: ["技术分享", "编程语言底层"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-14 · G1 的 SATB 写屏障与 RSet

## 📚 今日主题

> **G1 的 SATB 写屏障与 RSet**（编程语言底层）

### 1. 核心概念速览
SATB 写屏障（Snapshot At The Beginning Write Barrier）是 G1 垃圾收集器在并发标记阶段使用的一种写屏障，它记录被覆盖的旧引用值到 SATB 队列，以维护标记开始时的对象快照，防止并发赋值导致存活对象漏标。RSet（Remembered Set）是 G1 中每个 Region 关联的元数据结构，记录哪些其他 Region 的卡（Card）可能包含指向本 Region 内对象的引用，用于快速定位跨区域引用，避免全堆扫描。本质：SATB 是并发安全的标记协议，RSet 是空间换时间的分区引用索引。它们共同解决了 G1 在分区堆上实现并发标记和混合回收的正确性与效率问题。在计算机体系中的位置：属于 JVM 垃圾回收器实现层，是并发算法和内存管理的交叉点。专业工程师必须掌握，因为 G1 是 HotSpot 默认 GC，调优、排查内存泄漏和 STW 问题时，理解 SATB 和 RSet 是理解 GC 日志、停顿根因的基础。

### 2. 底层原理剖析
G1 将堆划分为多个 Region，采用并发标记 + 复制回收。并发标记阶段，GC 线程与业务线程并行执行。若业务线程修改了对象的引用，可能导致存活对象在标记完成后仍未被标记（漏标）。SATB 策略：标记开始时，所有存活对象构成一个逻辑快照；写屏障在所有引用赋值之前执行，将即将被覆盖的旧引用加入 SATB 队列。标记线程处理队列，标记这些旧引用指向的对象。这样，即使对象在并发期间被改写，其旧引用指向的对象仍被视为存活，保证不会漏标。代价是产生浮动垃圾（已死但被快照标记的对象）。
伪代码：
void satb_write_barrier(oop old_value) {
  if (old_value != NULL && !old_value->is_marked()) {
    satb_queue.enqueue(old_value);
  }
}
实际还会检查并发标记是否处于活动状态。
RSet 的维护：当引用 obj.field = new_ref 发生时，若 new_ref 与 obj 不在同一 Region，则写屏障（通常与 SATB 合并或独立）会将目标 Region 的 RSet 更新，记录源 Region 的卡索引。G1 采用卡表（Card Table）粗粒度标记脏卡，再通过并发线程扫描脏卡更新 RSet。RSet 以哈希表实现，键为源 Region 的卡索引，值为指向该卡的指针集合，去重后避免重复记录。回收某个 Region 时，遍历其 RSet，将外部引用作为 GC Roots，即可确定该 Region 内对象的可达性，无需扫描整个堆。
对比前端概念：JavaScript 引擎（如 V8）的分代 GC 中也有写屏障和 Remembered Set，用于记录老年代对象对新生代对象的引用，以便 Minor GC 时快速找到跨代引用。V8 的写屏障在对象字段写入时记录新引用（老->新），与 G1 的 SATB 记录旧引用不同，但两者的 RSet/Store Buffer 目的类似：避免全堆扫描。前端工程中的不可变数据（如 React 的 state）也蕴含快照思想，但 SATB 是内存安全的协议，不可变数据是设计模式，二者抽象层次不同。

### 3. 基础代码与实战验证
```text
以下为 G1 SATB 写屏障与 RSet 更新的核心逻辑伪代码（实际 HotSpot 实现更复杂，此处展示本质）：
// 引用赋值入口，obj 为被修改对象，offset 为字段偏移，new_value 为新引用
void write_reference(oop obj, size_t offset, oop new_value) {
    oop old_value = obj->field_at(offset);

    // —— SATB 写屏障 ——
    // 仅在并发标记阶段启用。记录旧值，而非新值。
    if (g1h->is_concurrent_marking_active()) {
        if (old_value != NULL && !old_value->is_marked()) {
            // 将旧值加入 SATB 队列，标记线程会扫描该队列
            satb_queue->enqueue(old_value);
        }
    }

    // 实际写入字段
    obj->field_at(offset) = new_value;

    // —— RSet 写屏障（后写屏障） ——
    // 若新引用指向的对象位于不同 Region，则更新目标 Region 的 RSet
    if (new_value != NULL) {
        Region* src_region = region_of(obj);
        Region* dst_region = region_of(new_value);
        if (src_region != dst_region) {
            // 将源 Region 中 obj 所在卡片标记为 dirty，并异步更新 dst_region->rset
            uint card_index = card_index_of(obj);
            dst_region->rset->add_reference(src_region, card_index);
        }
    }
}

// SATB 队列处理（标记线程）
void drain_satb_queue() {
    oop obj;
    while (satb_queue->dequeue(&obj)) {
        if (obj != NULL && !obj->is_marked()) {
            obj->set_marked();
            // 递归扫描其引用（省略）
        }
    }
}
注释说明：SATB 写屏障在赋值前捕获旧引用，保证快照；RSet 写屏障在赋值后记录跨区引用，便于 Region 回收时定位外部根。实际 JVM 中 RSet 更新有过滤和延迟批量处理机制，以减少写屏障开销。
```

### 4. 常见误区与进阶思考
误区1：认为 SATB 写屏障记录的是新引用值。实际上 SATB 记录的是被覆盖的旧引用，目的是保留标记开始时的快照。如果记录新值，无法防止并发期间旧对象被改写后漏标，且会引入更多遍历。理解这一点是区分 SATB 与增量更新的关键。
误区2：将 RSet 理解为对象级引用列表。RSet 是 Region 级，粒度是卡（Card），它记录的是“哪些源 Region 的哪些卡可能包含指向本 Region 的引用”，并非精确到具体对象。这种模糊性导致很多人误以为 RSet 是完整的引用图，实际上它是过滤后的近似集合，配合卡表使用。
思考题：在并发标记阶段，假设对象 A 已被标记，线程 T 执行 A.field = B，且 B 在此之前未被标记。SATB 写屏障会记录什么？B 会被标记吗？如果不会，G1 在后续回收中如何避免错误回收 B？请从 SATB 队列、RSet 和回收阶段的根集合角度分析。
