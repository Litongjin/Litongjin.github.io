---
title: "每日基础技术总结 · 2026-10-23 · V8 垃圾回收算法：Mark-Sweep 与 Mark-Compact 的分代收集策略"
date: 2026-10-23 07:02:06
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-23 · V8 垃圾回收算法：Mark-Sweep 与 Mark-Compact 的分代收集策略

## 📚 今日主题

> **V8 垃圾回收算法：Mark-Sweep 与 Mark-Compact 的分代收集策略**（前端底层与计算机基础）

### 1. 核心概念速览
**定义与本质**

V8 引擎的垃圾回收（Garbage Collection, GC）本质上是**自动内存管理**机制，其核心基于**可达性分析（Reachability Analysis）**算法。它通过从根对象（Root Set，包括全局对象、当前执行上下文栈、闭包变量等）出发，遍历对象引用图，标记所有可达对象，未被标记的即为“垃圾”，可被回收。这解决了手动内存管理（如 C/C++ 的 `malloc/free`）中常见的内存泄漏和悬垂指针问题。

V8 采用**分代收集（Generational Collection）**策略，基于“大部分对象生命周期很短”的**分代假说（Generational Hypothesis）**。内存被划分为**新生代（Young Generation，也称 New Space）**和**老生代（Old Generation，也称 Old Space）**。新生代使用 **Scavenge 算法**（一种复制式算法），而老生代则主要使用 **Mark-Sweep（标记-清除）** 和 **Mark-Compact（标记-整理）** 组合策略。此设计旨在通过差异化算法优化回收效率与内存利用率。

**在计算机体系中的位置**

V8 GC 是语言运行时（Runtime）的核心组件，位于 JavaScript 引擎内部，介于语言层（JS 代码）与操作系统内存管理层之间。它封装了底层的内存分配（如基于 `mmap` 或 `VirtualAlloc` 的系统调用）和释放细节，为 JS 开发者提供透明的内存管理。对于前端工程师而言，理解 V8 GC 是深入掌握性能优化、内存泄漏排查、以及 Node.js 服务端应用稳定性的基石。在 AI 体系中，类似机制也存在于 JVM（如 G1 GC）、CLR（.NET GC）等运行时，其思想对理解资源自动管理具有普适性。

**为什么专业工程师必须掌握**

1.  **性能优化**：GC 停顿（Stop-the-World）直接影响应用响应，尤其在高并发或实时性要求高的场景（如 WebGL 游戏、Node.js API 服务）。理解分代收集有助于通过对象生命周期管理减少 Full GC 频率。
2.  **内存泄漏诊断**：闭包、全局变量、未释放的定时器等导致的内存泄漏，本质上是对象被错误地标记为“可达”。掌握可达性分析是定位泄漏点的关键。
3.  **跨语言/平台迁移**：前端工程师拓展至后端（Node.js）或 AI 工程（如使用 Python 的 CPython、Java 的 JVM）时，GC 是共通的核心概念，其原理（如分代、标记-清除）在不同运行时中高度相似。
4.  **底层能力构建**：摒弃“黑盒”思维，理解 GC 能培养对内存模型、引用关系、系统调用的直觉，这是构建高可靠、高性能系统的底层基础。

### 2. 底层原理剖析
**底层运行机制剖析**

V8 GC 的核心流程可分为**标记（Marking）**、**清除（Sweeping）**、**整理（Compacting）**三个阶段，并结合分代策略执行。

**1. 分代内存布局与对象晋升**

*   **新生代（New Space）**：容量小（通常 1-8MB），存放存活时间短的对象。内部划分为两个等大的**半空间（Semispace）**：`From` 空间和 `To` 空间。对象分配始终在 `From` 空间进行。
*   **老生代（Old Space）**：容量大，存放存活时间长的对象。对象从新生代**晋升（Promotion）**而来，晋升条件：
    *   对象经历过一次 Scavenge 回收后仍存活。
    *   `To` 空间的使用率超过 25%（避免新生代频繁回收）。
    *   大对象（Large Object Space）直接分配在老生代，避免复制开销。

**2. 新生代 Scavenge 算法（复制式）**

伪代码描述：

    // 假设 From 空间已满，触发 Scavenge
    function scavenge() {
        // 1. 交换 From 和 To 空间角色
        swap(fromSpace, toSpace);
        // 2. 遍历根对象，标记存活对象并复制到 To 空间
        for (root in rootSet) {
            copyLiveObjects(root, toSpace);
        }
        // 3. 更新所有指向已移动对象的引用（指针重写）
        updateReferences();
        // 4. From 空间整体释放（无需逐个清除，效率高）
        free(fromSpace);
    }

**机制要点**：
*   **复制而非移动**：存活对象被复制到 `To` 空间，而非原地移动。这避免了内存碎片，但代价是始终有一半空间闲置。
*   **指针更新**：复制后，所有指向原对象的指针需更新为新地址，这涉及遍历对象图，是主要开销之一。
*   **适用场景**：适合对象生命周期短、存活率低的场景，因为复制成本与存活对象数量成正比，而非总对象数。

**3. 老生代 Mark-Sweep 与 Mark-Compact 组合策略**

当老生代空间不足或达到阈值时，触发 **Major GC**（也称 Full GC），采用 Mark-Sweep 为主，Mark-Compact 为辅的策略。

*   **Mark-Sweep（标记-清除）**
    *   **标记阶段**：从根对象出发，递归遍历所有可达对象，并在其头部（Header）设置标记位（Mark Bit）。V8 使用**三色标记（Tri-color Marking）**优化并发：白色（未访问）、灰色（已访问但子对象未扫描）、黑色（已访问且子对象已扫描）。
    *   **清除阶段**：遍历整个老生代堆，回收未标记（白色）对象的内存，并加入空闲列表（Free List）。**缺点**：会产生**内存碎片（Fragmentation）**，即不连续的小块空闲内存，可能导致后续大对象分配失败。

*   **Mark-Compact（标记-整理）**
    *   在 Mark-Sweep 基础上，增加**整理阶段**：将所有存活对象向堆的一端移动，紧凑排列，然后更新所有引用指针，最后释放边界外的内存。**优点**：消除碎片，但**缺点**是移动对象成本高，且需要暂停应用线程（Stop-the-World）时间更长。
    *   V8 的**组合策略**：并非每次 Major GC 都执行整理。而是先执行 Mark-Sweep，当碎片率达到阈值（如老生代空间不足以满足晋升对象分配）时，才触发 Mark-Compact。这种惰性整理（Lazy Compaction）平衡了效率与内存利用率。

**4. 并发与增量优化**

为减少停顿时间，V8 引入：
*   **增量标记（Incremental Marking）**：将标记过程拆分为小步骤，与 JS 执行交替进行，避免长时间停顿。
*   **并发标记（Concurrent Marking）**：在后台线程进行标记，进一步降低主线程阻塞。
*   **惰性清除（Lazy Sweeping）**：清除阶段按需进行，而非一次性遍历整个堆。

**与前端已有概念的异同对比**

| 特性                | V8 GC (JavaScript)                          | Java GC (如 G1)                              | 前端手动内存管理 (如 C/C++)          |
| :------------------ | :------------------------------------------ | :------------------------------------------- | :----------------------------------- |
| **核心算法**        | 分代收集（Scavenge + Mark-Sweep/Compact）   | 分代收集（G1 为 Region-based，可预测停顿）   | 手动分配/释放（`malloc/free`）       |
| **内存布局**        | 新生代（半空间）、老生代、大对象空间等      | Eden、Survivor、Old Generation、Humongous 区 | 堆（Heap）、栈（Stack）、静态区      |
| **对象晋升**        | 基于 Scavenge 存活次数和 To 空间阈值        | 基于年龄阈值（默认 15 次 Minor GC）          | 无自动晋升，完全手动控制             |
| **碎片处理**        | Mark-Compact 整理，或 Scavenge 复制         | G1 通过 Region 整理和 Mixed GC 减少碎片      | 手动管理，易产生碎片，需自定义分配器 |
| **停顿控制**        | 增量、并发标记，但仍有 Full GC 停顿风险     | G1 可设置最大停顿时间目标（如 200ms）        | 无停顿，但泄漏风险高                 |
| **开发者干预**      | 弱干预（如 `global.gc()` 暴露，但不推荐）   | 可通过 JVM 参数调优（如 `-Xmx`, `-XX:+UseG1GC`） | 完全手动，需配对使用 `new/delete`    |

**关键洞察**：V8 的 GC 策略在自动化与性能间取得平衡，其分代思想与 JVM 类似，但新生代采用复制式算法（Scavenge）而非 JVM 的复制-整理混合，更强调低停顿。前端工程师需从“内存是无限的”幻觉中觉醒，理解对象生命周期和引用关系，这与手动管理内存的严谨性一脉相承，只是自动化层屏蔽了细节。

### 3. 基础代码与实战验证
```text
以下代码演示如何在 Node.js 环境中观察 V8 GC 行为，验证分代收集和对象晋升机制。使用 `--expose-gc` 标志暴露全局 `gc()` 函数以手动触发 GC（仅用于学习，生产环境禁用）。

    // 文件：gc-demo.js
    // 运行：node --expose-gc gc-demo.js

    const v8 = require('v8');

    // 辅助函数：打印内存统计信息
    function printMemoryStats(label) {
        const stats = v8.getHeapStatistics();
        console.log(`--- ${label} ---`);
        console.log(`堆总大小: ${(stats.total_heap_size / 1024 / 1024).toFixed(2)} MB`);
        console.log(`已用堆大小: ${(stats.used_heap_size / 1024 / 1024).toFixed(2)} MB`);
        console.log(`新生代大小: ${(stats.young_generation_size / 1024 / 1024).toFixed(2)} MB`);
        console.log(`老生代大小: ${(stats.old_generation_size / 1024 / 1024).toFixed(2)} MB`);
        console.log('');
    }

    // 初始状态
    printMemoryStats('初始状态');

    // 1. 创建大量短命对象（应被新生代 Scavenge 回收）
    function createShortLivedObjects() {
        for (let i = 0; i < 100000; i++) {
            const obj = { data: new Array(100).fill(i) }; // 分配在新生代
            // obj 很快失去引用，成为垃圾
        }
    }

    createShortLivedObjects();
    printMemoryStats('创建短命对象后（未触发 GC）');

    // 手动触发新生代 GC（Scavenge）
    global.gc();
    printMemoryStats('手动 GC 后（新生代应被清理）');

    // 2. 创建长命对象，观察晋升到老生代
    const longLivedObjects = [];
    function createLongLivedObjects() {
        for (let i = 0; i < 5000; i++) {
            const obj = { id: i, data: new Array(1000).fill(i) };
            longLivedObjects.push(obj); // 保持引用，避免被回收
        }
    }

    createLongLivedObjects();
    printMemoryStats('创建长命对象后（部分可能晋升）');

    // 再次手动触发 Full GC（包括老生代 Mark-Sweep/Compact）
    global.gc();
    printMemoryStats('再次手动 GC 后（老生代稳定）');

    // 3. 模拟内存泄漏：闭包持有大对象
    function createLeak() {
        const largeData = new Array(1000000).fill('leak');
        return function() {
            // 闭包引用 largeData，导致无法回收
            return largeData.length;
        };
    }

    const leaks = [];
    for (let i = 0; i < 10; i++) {
        leaks.push(createLeak()); // 每次调用都创建新的大数组并被闭包持有
    }
    printMemoryStats('模拟闭包泄漏后');

    // 清理泄漏引用
    leaks.length = 0; // 切断引用链
    global.gc();
    printMemoryStats('清理泄漏引用并 GC 后');

**关键代码行注释**：

*   `v8.getHeapStatistics()`：V8 内置 API，返回堆内存详细统计，包括 `young_generation_size` 和 `old_generation_size`，用于观察分代变化。
*   `global.gc()`：手动触发 Full GC（包括新生代 Scavenge 和老生代 Major GC）。在 `--expose-gc` 下可用，用于验证回收行为。
*   `longLivedObjects.push(obj)`：保持对象引用，使其在 GC 后仍存活，可能晋升到老生代。观察 `old_generation_size` 增长。
*   `createLeak()`：闭包持有 `largeData`，即使外部函数执行完毕，`largeData` 仍被闭包引用，导致内存泄漏。这是前端常见的泄漏模式（如事件监听器未移除）。
*   `leaks.length = 0`：清空数组，切断对闭包的引用，使泄漏对象变为不可达，GC 可正常回收。

**验证要点**：
1.  运行后，观察“创建短命对象后”内存上升，但“手动 GC 后”应显著下降，验证新生代回收。
2.  “创建长命对象后”内存持续上升，且“再次手动 GC 后” `old_generation_size` 应稳定，说明对象已晋升并存活。
3.  “模拟闭包泄漏后”内存飙升，但“清理泄漏引用并 GC 后”应回落，验证可达性分析和闭包泄漏原理。
```

### 4. 常见误区与进阶思考
**常见认知误区**

1.  **误区一：认为“手动触发 `global.gc()` 可以优化生产环境性能”**
    *   **本质剖析**：V8 的 GC 调度基于复杂的启发式算法（如分配速率、堆增长趋势、历史停顿时间），旨在平衡吞吐量与延迟。手动调用 `gc()` 会强制同步执行 Full GC，包括完整的 Mark-Sweep/Compact，这会导致长时间的 **Stop-the-World** 停顿，严重阻塞事件循环。在 Node.js 高并发服务中，这可能引发请求超时、雪崩效应。正确做法是**通过对象生命周期管理减少 GC 压力**，例如：避免在热点路径创建大对象、使用对象池复用、及时解除引用（如 `null` 赋值）、谨慎使用闭包。生产环境应依赖 V8 的自动调度，仅在内存泄漏诊断或基准测试时临时使用 `--expose-gc`。

2.  **误区二：混淆“内存泄漏”与“内存使用高”，或认为“GC 能回收所有不再使用的对象”**
    *   **本质剖析**：GC 回收的依据是**可达性**，而非“是否被使用”。只要对象仍被根对象直接或间接引用（如全局变量、闭包、事件监听器、定时器回调、缓存未过期），即使业务逻辑上“不再使用”，GC 也不会回收。这导致**逻辑上的内存泄漏**。例如：
        *   单页应用（SPA）中，路由切换后未移除旧组件的事件监听器，导致 DOM 节点和关联对象无法释放。
        *   无界缓存（如 `Map` 持续增长）持有大量数据，即使数据已过期。
        *   闭包意外捕获大对象（如上述代码示例）。
    *   内存使用高可能是合理的（如大型数据处理），但持续增长且不回落才是泄漏信号。诊断需结合堆快照（Heap Snapshot）分析支配树（Dominator Tree）和保留路径（Retaining Path），而非仅凭内存曲线。

**进阶思考题**

**题目**：V8 的新生代 Scavenge 算法采用“复制式”设计，始终保留一半空间闲置；而老生代 Mark-Compact 则通过“整理”消除碎片，但移动对象成本高。假设你正在设计一个面向实时音视频处理（如 WebRTC）的 JavaScript 引擎，其中存在大量短命对象（如临时网络包）和少量长命对象（如编解码器状态）。

**要求**：
1.  从**内存碎片、复制开销、停顿时间**三个维度，分析为何 V8 选择“新生代复制 + 老生代整理”的差异化策略，而非统一使用同一种算法？
2.  如果让你优化此场景下的 GC，你会考虑引入哪种**增量式或并发式**改进？具体说明其如何与现有分代策略结合，并权衡其实现复杂度与收益。

**考察点**：是否理解分代假说的工程权衡、复制与整理的适用边界、以及并发 GC 的核心挑战（如写屏障、线程同步）。这检验能否将底层原理迁移至具体架构设计，而非仅记忆概念。
