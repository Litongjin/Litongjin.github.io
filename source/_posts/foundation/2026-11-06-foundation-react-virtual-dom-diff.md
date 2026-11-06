---
title: "每日基础技术总结 · 2026-11-06 · React 虚拟 DOM 与 Diff 算法"
date: 2026-11-06 10:52:17
categories: [技术分享]
tags: ["技术分享", "前端框架层（Vue / React / 工程化）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-06 · React 虚拟 DOM 与 Diff 算法

## 📚 今日主题

> **React 虚拟 DOM 与 Diff 算法**（前端框架层（Vue / React / 工程化））

### 1. 核心概念速览
## 1. 核心概念速览

**React Element（VNode）**：JSX 的编译产物（automatic runtime 下为 jsx(type, props, key)），形如 {type, key, ref, props}，是不可变的纯数据对象，不含宿主资源、不含生命周期、可被 GC 管理。它描述「某时刻 UI 应该是怎样的」，而非「如何变更 UI」。

**虚拟 DOM（VDOM）**：由 Element 组成的树形中间表示（IR）。其本质不在于「更快」，而在于把「命令式的宿主节点变更序列」降解为「两棵纯数据树的求差问题」，并把求差结果收敛为一组最小的写操作集合。

**Reconciliation / Diff**：在 current Fiber 树与 workInProgress Fiber 树之间，依据若干受限假设求出副作用集合（Placement / Update / Deletion / ChildDeletion）的算法；副作用不立即写宿主，而是冒泡到 flags，由 commit 阶段一次性提交。

**解决的问题**：
1. 状态到 UI 的映射不可逆——由新 state 无法反推最小 DOM 操作序列，故以「重新描述整棵树 + 求差」换取可证明的正确性与成本上界。
2. 宿主抽象——同一份 Element 树可提交至 react-dom / react-native / 自定义 host config。
3. 成本上界——通用树编辑距离（Zhang-Shasha）为 O(n^3)，带 key 的最短编辑序列为 O(n^2)；React 用「同层比较 + type 比较 + key 身份」把上界压缩到 O(n)，以理论最优性换取运行期可预测性。

**体系位置**：渲染管线「Scheduler（调度）→ Reconciler（协调，可中断）→ Commit（提交，同步）」的中段。Fiber 把 Diff 从不可中断的递归函数改造为可中断、可恢复的工作单元链表，是 Concurrent Rendering、Suspense、时间切片、Hooks 状态链表（fiber.memoizedState）得以存在的前提。不理解 diff 与 flag 传播，就无法解释：setState 后 DOM 何时真正变化、useLayoutEffect 为何在 mutation 之后 layout 之前执行、key 变化为何等价于卸载重建。

### 2. 底层原理剖析
## 2. 底层原理剖析

### 2.1 三条启发式假设与复杂度来源
1. 不同类型的元素产生不同的子树 → 不做跨层移动检测，命中差异即整棵卸载重建。
2. 只做同层比较（同一父节点下的 children 数组）→ 树 diff 退化为「每层做一次列表 diff」。
3. key 由开发者提供，用于在同层兄弟之间声明稳定身份。

缺失 key 时按 index 匹配：这是**身份契约失效**，先破坏的是正确性（状态归属），其次才是性能。

### 2.2 单节点
old.type === new.type 且 old.key === new.key → 复用宿主实例，仅 patch 差异属性（可进一步由 props 相等触发 bailout）；否则 deleteRemainingChildren + 新建。

### 2.3 列表 Diff（reconcileChildrenArray）
两个 pass：
- **pass1**：i 从 0 同步遍历 old/new，按 key（无 key 退化为 index）比对；命中则 patch 并更新 lastPlacedIndex；一旦不命中立即 break（剩余旧节点可能整体后移）。
- **pass2**：把剩余 old children 全部放入 Map（key → fiber）；遍历剩余 new children，Map 命中则复用并按 lastPlacedIndex 判定是否移动，否则新建；遍历结束后 Map 中残留的旧 fiber 全部标记 Deletion。

**lastPlacedIndex 精确语义**：设 oldIdx 为命中的旧 fiber 在原列表中的下标。若 oldIdx < lastPlacedIndex，说明该节点相对顺序已被打乱，标记 Placement（需要 DOM 移动）；否则 lastPlacedIndex = oldIdx，不移动。这是一个贪心近似，保证「已处于正确相对位置的节点不再移动」，**不保证移动次数最少**（Vue 3 用最长递增子序列求最小移动集合）。

### 2.4 双缓冲与提交
- fiber.alternate 指向另一棵树；render 阶段全程在 workInProgress 上构建，current 树始终保持对外一致（可中断、可丢弃）。
- 副作用以 flags（Placement / Update / Deletion / ChildDeletion）冒泡至父链，commit 阶段三遍扫描（BeforeMutation / Mutation / Layout）一次性执行宿主写。这是「React 更新批量、可中断、可重放」的结构性原因。
- Mutation 阶段执行 Placement 的语义是 parentNode.insertBefore(child, before)：DOM 移动天然是「移动到新位置」，因此只需保证相对顺序正确，最小移动数并非必要——这是 React 不采用 LIS 的根本理由。

### 2.5 与前端已有知识体系的对照
- **JSX vs TypeScript 类型**：两者都在编译期处理，方向却相反。TS 类型在 emit 后被完全擦除（type erasure），运行期零字节；JSX 被编译为运行期真实对象，可比较、可序列化、可跨平台传输。对照 Java：Java 的接口与注解在 class 文件中保留，运行期可反射读取、调用经 itable/vtable 分派——React 选择了「运行期保留结构化描述」的路线，代价是每次更新产生对象分配，收益是平台无关与可 diff。
- **与编译器 IR 同构**：VNode 树 ≈ 前端 IR，diff ≈ IR 上的规范化 pass。Vue 3 的 Block Tree 与 PatchFlag、React Compiler 的自动 memo 化，本质都是把 diff 从运行期前移到编译期（消除比较，而非加速比较），与 JIT 的常量折叠同构。
- **与原生 DOM 的边界**：Blink 的 DOM 节点是 C++ 对象，JS 侧经 wrapper 跨界访问；attribute 与 property 语义分裂，样式重算与布局存在 layout/paint 成本。VNode diff 将所有「读」搬到 JS 堆上完成，跨界仅剩最终的写集合。

### 3. 基础代码与实战验证
```text
// ---- 1. JSX 的运行期形态：纯数据，无宿主资源，无行为 ----
function h(type, props, ...children) {
  return { type, props: props || {}, key: props && props.key, children: children.flat(), el: null };
}

// ---- 2. 首次挂载：VNode -> 宿主节点，仅在创建路径上跨界 ----
function createDom(vnode) {
  const el = document.createElement(vnode.type);
  vnode.el = el;                                   // 回填引用：后续 diff 依赖它定位宿主节点
  for (const k in vnode.props) if (k !== 'key') el.setAttribute(k, vnode.props[k]);
  for (const c of vnode.children) el.appendChild(createDom(c));  // 递归下降，只增不减
  return el;
}

// ---- 3. 属性 patch：先删后设，求出最小写集合，避免全量覆盖 ----
function patchProps(el, prev, next) {
  for (const k in prev) if (k !== 'key' && !(k in next)) el.removeAttribute(k);
  for (const k in next) if (k !== 'key' && prev[k] !== next[k]) el.setAttribute(k, next[k]);
}

// ---- 4. 单节点 diff：类型相同才复用宿主实例，否则整棵子树重建 ----
function patch(oldV, newV, parent, anchor) {
  if (!oldV) { parent.insertBefore(createDom(newV), anchor); return; }   // 新增路径
  if (oldV.type !== newV.type) {                                         // 类型不同：卸载 + 重建，旧子树状态丢弃
    parent.replaceChild(createDom(newV), oldV.el); return;
  }
  const el = (newV.el = oldV.el);                                        // 复用宿主实例，保留 DOM 内部状态
  patchProps(el, oldV.props, newV.props);
  diffChildren(oldV.children, newV.children, el);                        // 递归同层比较
}

// ---- 5. 列表 diff：key 建 Map + 从尾部向前 insertBefore ----
function diffChildren(oldCh, newCh, parent) {
  const map = new Map();
  oldCh.forEach((c, i) => map.set(c.key != null ? c.key : '#' + i, c));  // 身份 = key，缺失时降级为 index
  const used = new Set();
  let anchor = null;                                                     // 从后往前插，天然维持顺序
  for (let i = newCh.length - 1; i >= 0; i--) {
    const nv = newCh[i], k = nv.key != null ? nv.key : '#' + i;
    const ov = map.get(k);
    if (ov && ov.type === nv.type) {
      used.add(k);
      patch(ov, nv, parent, anchor);                                     // 先让子树完成自身 diff
      if (nv.el.nextSibling !== anchor) parent.insertBefore(nv.el, anchor); // 顺序纠正；生产版由 lastPlacedIndex 判定必要性
    } else {
      parent.insertBefore(createDom(nv), anchor);                        // 无匹配：新建并插入
    }
    anchor = nv.el;
  }
  map.forEach((ov, k) => { if (!used.has(k)) parent.removeChild(ov.el); }); // Map 残留即为被删除节点
}

// ---- 验证：a,b,c -> c,a,b，仅 c 前移；a 下的 input 值与焦点不丢失 ----
const t1 = h('ul', {}, h('li', { key: 'a' }, h('input', {})), h('li', { key: 'b' }), h('li', { key: 'c' }));
const t2 = h('ul', {}, h('li', { key: 'c' }), h('li', { key: 'a' }, h('input', {})), h('li', { key: 'b' }));
const root = document.body;
root.appendChild(createDom(t1));   // 首次挂载：建立 VNode 与宿主节点的双向引用
patch(t1, t2, root, null);         // 同层 diff：a 的宿主 li/input 被复用，仅发生 1 次移动
// 对照：把 key 换成 index，a 位置的 li 会与 c 的旧 fiber 错配，input 状态被移到错误节点上
```

### 4. 常见误区与进阶思考
## 4. 常见误区与进阶思考

### 误区一：虚拟 DOM 更快
Diff 是**新增**的纯计算成本，而量级更大的成本在宿主侧：样式重算、布局、绘制。VDOM 的收益是把「读」搬到 JS 堆上、把「写」压缩为最小集合、并让开发者免于维护变更序列；在更新路径已知且细粒度（动画、拖拽、Canvas 类）时，直接操作 DOM/属性通常更快。澄清：通用树编辑距离是 O(n^3)，React 的 O(n) 是三条启发式假设的**产物**而非事实——key 缺失或不稳定时，算法退化为按 index 匹配，先被破坏的是状态归属的正确性，而非性能。

### 误区二：key 只用于消除 warning，用 index 等价
key 是 fiber 的**身份契约**，同时决定三件事：宿主实例复用、组件实例（fiber.stateNode）与 Hooks 链表的归属、effect 的 destroy/recreate 时机。以 index 作 key，在头部插入、中间删除、排序、过滤时会把旧 fiber 错配给新的逻辑实体，导致 useState 状态错位、input 光标与 IME 组合被打断、transition 从错误初值启动；性能上，乱序更新也会从 O(1) 次 move 退化为 O(n) 次 patch。

### 进阶思考题
设旧列表 key = [a, b, c, d]，新列表 key = [d, c, b, a]。请推导：React 的 lastPlacedIndex 算法会产生多少次 DOM 移动？Vue 3 的 LIS 方案产生多少次？并论证 React 为何接受这个非最优解——从 insertBefore 对已挂载节点的语义、MutationObserver 与布局成本、以及「最坏情况上界 vs 平均情况常数」三个角度给出结论。（提示：两者的移动次数上界同为 O(n)，差别在常数与实现复杂度；而当 key 从 index 退化为隐式身份时，上述讨论全部失效。）
