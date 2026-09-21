---
title: "每日基础技术总结 · 2025-01-04 · 二分查找：边界处理与变种"
date: 2025-01-04 20:00:00
categories: [技术分享]
tags: ["技术分享", "算法与数据结构（面试）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2025-01-04 · 二分查找：边界处理与变种

## 📚 今日主题

> **二分查找：边界处理与变种**（算法与数据结构（面试））

### 1. 核心概念速览
二分查找（Binary Search）是一种在有序数组中通过每次迭代将搜索空间减半来定位目标值的分治算法。其本质是利用数据的单调性（Monotonicity），将线性时间的 $O(N)$ 比较降为对数时间的 $O(	ext{log}N)$。在 AI 体系中，它是数值计算、梯度下降初始化及大规模数据检索的基础原语。专业工程师必须掌握它，因为它是理解更复杂数据结构（如跳表、B+树）和优化工程性能的第一块基石，且极易因边界条件错误导致死循环或越界。

核心机制：维护一个闭区间或半开区间，根据中间元素与目标值的大小关系，收缩有效搜索范围，直到范围为空或找到目标。

前端对比：类似于 React 虚拟 DOM Diff 算法中的深度/广度优先遍历逻辑简化版，但二分查找严格依赖数据的有序性这一前置约束，而非结构的可比性。

### 2. 底层原理剖析
底层运行机制依赖于索引算术与区间不变式（Loop Invariant）。

1. 区间定义：通常采用左闭右开 [left, right) 或左右皆闭 [left, right]。选择决定更新规则。
2. 中点计算：mid = left + (right - left) >> 1。使用位移运算避免整数溢出，这是工程实现的底线。
3. 状态转移：
   - 若 arr[mid] < target: 目标在右侧，left 移至 mid + 1（左闭右开）或 mid + 1（左右皆闭）。
   - 若 arr[mid] > target: 目标在左侧，right 移至 mid（左闭右开）或 mid - 1（左右皆闭）。
   - 若 arr[mid] == target: 返回 mid 或继续向特定方向收缩以寻找首次/末次出现位置。
4. 终止条件：当 left >= right 时，区间为空，搜索结束。

与前端的异同：前端 TypeScript 的类型推断是静态的、基于结构子类型的；而二分查找的‘边界’是动态的、基于数值大小的时序约束。TS 接口要求形状兼容，二分查找要求状态一致（Consistency），即每次迭代后，如果解存在，必在当前区间内。

### 3. 基础代码与实战验证
```text
/**
 * 寻找第一个大于等于 target 的位置（Lower Bound）
 * 采用左闭右开区间 [left, right)
 */
function lowerBound(arr, target) {
    let left = 0;
    let right = arr.length; // 初始右边界为长度，体现左闭右开特性

    while (left < right) {
        // 防止 overflow 的位运算写法，等价于 Math.floor((left + right) / 2)
        const mid = left + ((right - left) >> 1);

        if (arr[mid] < target) {
            // arr[mid] 不可能是答案，排除 mid 及其左侧所有元素
            left = mid + 1;
        } else {
            // arr[mid] >= target，mid 可能是答案，也可能是更左侧的答案
            // 因此保留 mid，向右缩小左边界，但不能跳过 mid
            right = mid;
        }
    }

    // 循环结束时 left == right，即为插入点或首个满足条件的索引
    return left;
}
```

### 4. 常见误区与进阶思考
1. 整数溢出：在大型语言或高版本 C/C++ 中，(left + right) / 2 可能超出整型最大值导致未定义行为。必须使用 left + (right - left) / 2 或位运算。
2. 死循环陷阱：在使用左右皆闭区间 [left, right] 且 mid = floor((left + right) / 2) 时，若更新规则写为 left = mid 或 right = mid，当 left = mid, right = mid + 1 时，mid 仍等于 left，导致无限循环。解决方式包括改用 ceiling 除法取上整，或严格调整边界更新逻辑（如 left = mid + 1）。

思考题：如果数组中存在重复元素，如何修改标准二分查找逻辑，使其能在 O(log N) 时间内分别找到 target 第一次出现和最后一次出现的索引？请从区间收缩策略的角度阐述，为什么简单的相等判断返回不够用？
