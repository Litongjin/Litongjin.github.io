---
title: "每日基础技术总结 · 2026-09-18 · 红黑树插入修复的三种情况与旋转"
date: 2026-09-18 08:00:00
categories: [技术分享]
tags: ["技术分享", "算法与数据结构"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-18 · 红黑树插入修复的三种情况与旋转

## 📚 今日主题

> **红黑树插入修复的三种情况与旋转**（算法与数据结构）

### 1. 核心概念速览
红黑树是满足五条不变量的二叉搜索树：节点红/黑；根黑；NIL 叶黑；红节点的子节点必黑；任一节点到其 NIL 后代的路径黑节点数相同（黑高相等）。插入修复：先按 BST 插入新节点并着红，此时唯一可能违反的是“红节点子节点黑”（性质4）。设 z 为冲突节点，P=z.p，G=z.p.p，y 为叔节点。修复按 y 的颜色与 z 相对 P 的位置分三种情况（左/右完全镜像）：Case1 y 红：P、y 置黑，G 置红，z=G，冲突可能上移；Case2 y 黑且 z 是 P 的右子（P 为 G 左子时）：对 P 左旋，z=P，转为 Case3；Case3 y 黑且 z 是 P 的左子：P 置黑，G 置红，对 G 右旋，修复终止。机制是旋转保持 BST 中序不变，重着色调整黑高；插入最坏 O(log n)，至多 2 次旋转。位置：动态有序集合、内存索引、Linux CFS、Nginx timer、std::map/Java TreeMap 等。前端常依赖哈希表与数组排序，红黑树提供确定 O(log n) 与有序/范围查询；掌握它是理解平衡树不变量、源码与后端/AI 基础设施的基础。

### 2. 底层原理剖析
不变量与目标：插入前树合法。新节点着红，只可能违反性质4。修复循环 while P.color == RED。因根黑，P 红时 G 必存在且为黑。以下以 P == G.left 为例，右子对称。
伪代码：
while z.p.color == RED:
  if z.p == z.p.p.left:
    y = z.p.p.right
    if y.color == RED:                // Case 1
      z.p.color = BLACK; y.color = BLACK; z.p.p.color = RED; z = z.p.p
    else:
      if z == z.p.right:              // Case 2
        z = z.p; rotateLeft(z)
      z.p.color = BLACK;              // Case 3
      z.p.p.color = RED; rotateRight(z.p.p)
  else: 左右镜像
root.color = BLACK

Case1 本质：P 与 y 均为红，G 为黑。P、y 变黑，G 变红。从 G 出发到左右 NIL 的路径黑高保持不变，但 G 变红可能与 G.p 冲突，故 z=G 向上继续。Case2 本质：y 黑且 z 为右子，左旋 P 不改变颜色，只把“右子形态”转成“左子形态”，使 z 变为左子，转入 Case3。Case3 本质：y 黑且 z 为左子，P 置黑、G 置红后对 G 右旋。旋转后 P 成为子树根且为黑，红红冲突消除；从 P 到所有 NIL 的黑高与原 G 子树一致，故修复终止。
旋转：左旋 x 使 x 的右子 y 上移，x 成为 y 的左子，y 的左子挂到 x 的右子；右旋对称。旋转只改 O(1) 指针，保持 BST 中序序列。
与前端对比：类似虚拟 DOM diff 中通过 key 复用、移动节点维持 UI 结构，但 diff 是启发式且不保证全局有序；红黑树修复是确定性不变量维护，旋转对应 AST/链表中的指针重连。TS 类型收窄是编译期约束，红黑树是运行期结构约束。

### 3. 基础代码与实战验证
```text
const RED = 1, BLACK = 0;
class Node {
  constructor(key) { this.key = key; this.color = RED; this.left = null; this.right = null; this.p = null; }
}
class RBTree {
  constructor() { this.nil = new Node(null); this.nil.color = BLACK; this.root = this.nil; }
  rotateLeft(x) {
    const y = x.right;
    x.right = y.left;
    if (y.left !== this.nil) y.left.p = x;
    y.p = x.p;
    if (x.p === this.nil) this.root = y;
    else if (x === x.p.left) x.p.left = y;
    else x.p.right = y;
    y.left = x;
    x.p = y;
  }
  rotateRight(x) {
    const y = x.left;
    x.left = y.right;
    if (y.right !== this.nil) y.right.p = x;
    y.p = x.p;
    if (x.p === this.nil) this.root = y;
    else if (x === x.p.right) x.p.right = y;
    else x.p.left = y;
    y.right = x;
    x.p = y;
  }
  insert(key) {
    const z = new Node(key);
    z.left = z.right = this.nil;
    let y = this.nil, x = this.root;
    while (x !== this.nil) { y = x; x = z.key < x.key ? x.left : x.right; }
    z.p = y;
    if (y === this.nil) this.root = z;
    else if (z.key < y.key) y.left = z;
    else y.right = z;
    z.color = RED; // 新节点必须红：插入黑会立即破坏所有路径黑高一致
    this.fixup(z);
  }
  fixup(z) {
    while (z.p.color === RED) {
      if (z.p === z.p.p.left) {
        const y = z.p.p.right; // 叔节点
        if (y.color === RED) { // Case 1：叔红，仅重着色，祖父变红可能上移冲突
          z.p.color = BLACK;
          y.color = BLACK;
          z.p.p.color = RED;
          z = z.p.p;
        } else {
          if (z === z.p.right) { // Case 2：叔黑且 z 为右子，左旋父转 Case 3
            z = z.p;
            this.rotateLeft(z);
          }
          z.p.color = BLACK; // Case 3：叔黑且 z 为左子，父黑祖父红后右旋祖父
          z.p.p.color = RED;
          this.rotateRight(z.p.p);
        }
      } else {
        const y = z.p.p.left;
        if (y.color === RED) {
          z.p.color = BLACK;
          y.color = BLACK;
          z.p.p.color = RED;
          z = z.p.p;
        } else {
          if (z === z.p.left) {
            z = z.p;
            this.rotateRight(z);
          }
          z.p.color = BLACK;
          z.p.p.color = RED;
          this.rotateLeft(z.p.p);
        }
      }
    }
    this.root.color = BLACK; // 根黑是最终保证
  }
  inorder() {
    const out = [];
    const dfs = (n) => {
      if (n !== this.nil) { dfs(n.left); out.push(n.key); dfs(n.right); }
    };
    dfs(this.root);
    return out;
  }
}
const t = new RBTree();
[7,3,18,10,22,8,11,26,2,6,13].forEach(k => t.insert(k));
console.log(t.inorder()); // 升序：[2,3,6,7,8,10,11,13,18,22,26]
```

### 4. 常见误区与进阶思考
误区1：把 Case 1 当作需要旋转。Case 1 的条件是叔节点红，处理方式只有重着色，不旋转；旋转只发生在叔黑时的 Case 2/3。Case 1 将 z 上移到 G，可能连续多次，但旋转次数仍为常数。
误区2：混淆 Case 2 与 Case 3 中 z 的指向。Case 2 左旋 P 后，z 应指向原 P（旋转后的左子），而不是原 z；否则 Case 3 的 z.p、z.p.p 全部错位，后续变色和旋转对象错误。另一个常见错误是认为新节点可以着黑，这会在插入点立即破坏所有路径黑高相等，需要更复杂的全局调整。
深度思考题：在 Case 3 中，父 P 置黑、祖父 G 置红后对 G 右旋。请从旋转前后子树内每条到 NIL 的路径黑节点数变化，证明为什么旋转后从新子树根 P 到所有 NIL 的黑高与原 G 子树一致，且红红冲突一定终止，不需要继续向上循环。若 P 的父为红，为什么此时也不会再违反性质4？
