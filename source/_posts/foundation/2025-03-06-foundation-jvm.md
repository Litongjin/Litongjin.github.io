---
title: "每日基础技术总结 · 2025-03-06 · JVM 垃圾回收：分代收集与三色标记"
date: 2025-03-06 20:00:00
categories: [技术分享]
tags: ["技术分享", "Java 后端与 Spring 生态"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2025-03-06 · JVM 垃圾回收：分代收集与三色标记

## 📚 今日主题

> **JVM 垃圾回收：分代收集与三色标记**（Java 后端与 Spring 生态）

### 1. 核心概念速览
分代收集（Generational Collection）基于弱分代假说：绝大多数对象朝生夕死。JVM将堆内存划分为新生代和老年代，对新生代采用高频率的Copying算法（如Eden/Survivor区复制），对老年代采用Mark-Sweep-Compact或标记-清除算法以最小化停顿。三色标记法（Three-Color Marking）是并发标记阶段的原子性保障机制：黑色表示已扫描且引用完整，灰色表示正在扫描但存在未处理子引用，白色表示未访问。该机制解决并发环境下对象状态变更导致的漏标（Liveness Error）或错标（Correctness Error）问题，是Java高吞吐低延迟GC策略的核心基石。

### 2. 底层原理剖析
1. 分代策略：新生代对象在Eden分配，Minor GC后存活对象移入Survivor区并经历年龄增长，达到阈值（默认15）或Survivor溢出则晋升老年代。此过程通过指针碰撞（TLAB）或空闲列表优化分配效率。
2. 三色标记并发安全：
   - 初始增量标记：建立Root集合，遍历可达对象着色。
   - 并发修改（Write Barrier）：当黑色对象A引用白色对象B被修改时，触发写屏障（SATB或G1Humongous）。若新引用指向白色对象，将其压入重新扫描栈；若移除旧引用，需检查旧引用是否还指向其他灰色/黑色节点。
   - 饱和标记与并发重置：直到无灰色节点，进行最终同步阶段确保一致性。
3. 前端对比：TS的接口是编译时契约（Static Contract），仅约束结构形状；Java接口是运行时多态机制（Runtime Polymorphism）+ JVM方法表分派。同理，TS的类型守卫是静态分析，而JVM的GC策略是动态内存布局管理，前者关注代码逻辑正确性，后者关注运行时资源生命周期管理。

### 3. 基础代码与实战验证
```text
/**
 * 演示分代晋升与轻量级GC触发
 * 注意：实际GC行为受-Xmx, -XX:MaxTenuringThreshold等参数控制
 */
public class GenerationalDemo {
    public static void main(String[] args) {
        // Eden区分配
        Object youngObj = new Object(); 
        
        // 模拟多次GC循环，观察晋升机制
        // 在JVM日志中可见 'Minor GC' (Copy Algorithm)
        for (int i = 0; i < 100; i++) {
            new Object(); // 触发Eden满时的Minor GC
            System.gc();  // 显式请求Full GC (慎用，通常由Promotion Failure触发)
        }
        
        // 此时youngObj可能因年龄不足仍在新生代，或已晋升老年代
        // 此处不引用youngObj，其成为GC Roots外的孤岛
        youngObj = null;
        System.out.println("Demo End");
    }
}/*
 * 伪代码：写屏障(SATB)逻辑核心
 * if (old_ref instanceof White) {
 *     push_to_scan_stack(old_ref);
 * }
 */
```

### 4. 常见误区与进阶思考
1.误区：认为调用System.gc()能精确控制垃圾回收时机或代际。事实：JVM决定是否执行Full GC受多种启发式算法影响，强制System.gc()往往导致全局STW停顿，性能灾难。应通过调整堆大小和GC参数调优。
2.误区：混淆并发标记的安全性与线程安全。三色标记解决的只是‘可见性’一致性问题，防止对象被回收却仍被引用（Use-After-Free）或可到达对象被错误回收。它不解决Java语言层面的变量竞态条件（Race Condition）。
思考题：在G1 GC的Mixed GC过程中，如果并发标记阶段检测到对象引用的变化（如对象从老年代移动到新生代区域，即Region Move），GC线程如何处理这种内存布局的动态重排带来的并发安全性保证？
