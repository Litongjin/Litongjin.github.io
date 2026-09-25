---
title: "每日基础技术总结 · 2026-09-11 · 红黑树的旋转操作与插入/删除后的自平衡逻辑"
date: 2026-09-11 08:00:00
categories: [技术分享]
tags: ["技术分享", "算法与数据结构"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-11 · 红黑树的旋转操作与插入/删除后的自平衡逻辑

## 📚 今日主题

> **红黑树的旋转操作与插入/删除后的自平衡逻辑**（算法与数据结构）

### 1. 核心概念速览
红黑树是一种自平衡二叉查找树，每个节点增加一位存储颜色（红或黑），通过对任意根到叶子的路径上节点着色的约束，保证最长路径不超过最短路径的两倍，从而确保插入、删除、查找操作的时间复杂度为 O(log n)。其本质是“用颜色标记的 2-3-4 树的二叉化表示”，旋转与变色正是对 2-3-4 树节点分裂与合并的模拟。红黑树是工程中应用最广的平衡树之一（如 Linux 内核的 rbtree、Java TreeMap、C++ std::map），它比 AVL 树在插入/删除时重平衡次数更少，适合频繁写操作的场景。专业工程师必须掌握它，因为它是理解分布式系统、数据库索引、内存管理、并发容器等底层机制的基本功，也是面试中考察算法功底与严谨思维的经典载体。

### 2. 底层原理剖析
红黑树五条性质：1) 节点非红即黑；2) 根为黑；3) 所有叶子（NIL）为黑；4) 红节点的子节点必须为黑（即红节点的父节点不能为红）；5) 从任意节点到其每个叶子节点的所有路径包含相同数目的黑节点（黑高相等）。

旋转是保持二叉搜索树中序性质的局部结构调整，分为左旋和右旋。左旋以某个节点 x 为轴心，将其右孩子 y 提升为父节点，x 变为 y 的左孩子，y 的原左子树转为 x 的右子树。右旋对称。旋转只改变 O(1) 个指针，不改变节点颜色，但会改变局部子树的黑高分布，是修复性质的原子操作。

插入后的自平衡逻辑（以新插入节点 z 为红色开始，保证性质5不被破坏，只需修复性质4）：
- 情形0：z 是根，直接染黑。
- 情形1：z 的叔节点为红。父节点和叔节点变黑，祖父节点变红，然后将 z 上移为祖父节点继续循环。本质是 2-3-4 树的 4-节点分裂：将中间键提升到父层并染红。
- 情形2：z 的叔节点为黑，且 z 是内侧插入（即父节点是祖父的左孩子，z 是父节点的右孩子；或对称）。对父节点做一次旋转，使其变成情形3。
- 情形3：z 的叔节点为黑，且 z 是外侧插入（父和 z 同侧）。对祖父节点旋转，父节点与祖父节点颜色交换（父变黑，祖父变红）。此时子树根为黑，且黑高恢复，循环终止。

删除后的自平衡逻辑复杂度更高，核心是“借黑色”或“推黑色”。删除节点后若被删节点或替代者是黑色，则导致某路径黑高少 1，记当前节点 x（替代者）为“双重黑”。修复循环：
- 情形0：x 为根，直接染黑结束。
- 情形1：x 的兄弟节点 w 为红。交换 w 和父节点颜色，然后对父节点旋转，使兄弟变为黑，转化为后续情形。
- 情形2：w 为黑，且 w 的两个孩子都为黑。将 x 的黑上移，即 x 减一重黑，w 变红，x 上移为父节点，继续循环。本质是父节点吸收黑色，并把双重黑向上传递。
- 情形3：w 为黑，w 的左孩子红、右孩子黑（对称情形）。交换 w 与其左孩子颜色，对 w 右旋，使情形转为4。
- 情形4：w 为黑，w 的右孩子红。把父节点颜色赋给 w，父节点变黑，w 的右孩子变黑，对父节点左旋，x 置为根，结束。本质是借兄弟子树的红色节点来补足黑色，同时保持局部黑高。

与前端已有知识的对比：前端工程师熟悉的 CSS 中“flex 布局的自动换行与弹性伸缩”、React 的协调（diff）算法都是基于约束的局部调整；红黑树的旋转类似于 DOM 中“transform: rotate”只改变视觉位置而不改变元素层级属性，旋转前后中序遍历顺序不变；而颜色约束类似于 CSS 选择器优先级或 TypeScript 类型收窄：性质4和5是全局不变量，局部冲突必须通过有限模式（情形）归纳修复，这和 CSS 层叠样式规则需要寻找最长前缀匹配或 TS 中类型兼容性的可判定规则类似。不同点在于红黑树的修复是确定性的、可证明的，而前端布局或状态更新往往事启发式的。

### 3. 基础代码与实战验证
```text
以下为红黑树插入和删除后修复的核心伪代码（无框架依赖，C 风格结构体）：

typedef enum Color { RED, BLACK } Color;
typedef struct RBNode {
    int key;
    Color color;
    struct RBNode *left, *right, *parent;
} RBNode;

// 左旋：将以 x 为根的子树向右旋转，x 的右孩子 y 成为子树根
void leftRotate(RBNode **root, RBNode *x) {
    RBNode *y = x->right;          // y = x 的右孩子
    x->right = y->left;            // y 的左子树交给 x 作为右子树
    if (y->left != NULL) y->left->parent = x;
    y->parent = x->parent;         // y 接管 x 的父指针
    if (x->parent == NULL)
        *root = y;                 // x 曾是根，更新根
    else if (x == x->parent->left)
        x->parent->left = y;
    else
        x->parent->right = y;
    y->left = x;                   // x 变成 y 的左孩子
    x->parent = y;
}
// 右旋对称：y = x->left，逻辑镜像。

// 插入修复：循环处理红色父节点冲突
void insertFixup(RBNode **root, RBNode *z) {
    while (z->parent != NULL && z->parent->color == RED) {
        if (z->parent == z->parent->parent->left) {   // 父是祖父左孩子
            RBNode *uncle = z->parent->parent->right; // 叔在祖父右
            if (uncle != NULL && uncle->color == RED) { // 情形1：叔红，变色上浮
                z->parent->color = BLACK;
                uncle->color = BLACK;
                z->parent->parent->color = RED;
                z = z->parent->parent;                // 将冲突上移两层
            } else {                                  // 叔黑，需旋转
                if (z == z->parent->right) {          // 情形2：内测插入
                    z = z->parent;
                    leftRotate(root, z);              // 先左旋变成外围
                }
                z->parent->color = BLACK;             // 情形3：外侧插入，换色+右旋
                z->parent->parent->color = RED;
                rightRotate(root, z->parent->parent);
            }
        } else { /* 对称逻辑：父是祖父右孩子，镜像操作 */ }
    }
    (*root)->color = BLACK;                           // 根始终染黑
}

// 删除修复：x 是删除后实际接替的节点，标记为额外黑
void deleteFixup(RBNode **root, RBNode *x) {
    while (x != *root && (x == NULL || x->color == BLACK)) {
        if (x == x->parent->left) {
            RBNode *w = x->parent->right;             // w 为兄弟
            if (w->color == RED) {                    // 情形1：兄弟红，转成兄弟黑
                w->color = BLACK;
                x->parent->color = RED;
                leftRotate(root, x->parent);
                w = x->parent->right;                 // 更新兄弟
            }
            if ((w->left == NULL || w->left->color == BLACK) &&
                (w->right == NULL || w->right->color == BLACK)) {
                w->color = RED;                       // 情形2：两侄黑，x 上推一层
                x = x->parent;
            } else {
                if (w->right == NULL || w->right->color == BLACK) {
                    // 情形3：右侄黑，左侄红，转成情形4
                    if (w->left != NULL) w->left->color = BLACK;
                    w->color = RED;
                    rightRotate(root, w);
                    w = x->parent->right;
                }
                // 情形4：右侄红，借红补黑后终止
                w->color = x->parent->color;
                x->parent->color = BLACK;
                if (w->right != NULL) w->right->color = BLACK;
                leftRotate(root, x->parent);
                x = *root;
            }
        } else { /* 对称镜像 */ }
    }
    if (x != NULL) x->color = BLACK;
}
```

### 4. 常见误区与进阶思考
常见误区1：认为红黑树只是“带颜色的二叉搜索树”，旋转是随意的。实际上旋转必须严格遵循情形分支，且每次旋转后必须保持中序有序性，错误的旋转顺序或颜色交换会导致黑高失衡，甚至破坏 BST 性质。很多实现中容易漏掉叔节点为 NULL 的情况（NULL 视为黑），导致对空指针取颜色造成崩溃。

常见误区2：误以为红黑树比 AVL 树更平衡。红黑树的平衡是“弱平衡”（最长路径不超过最短两倍），其优势在于插入/删除的旋转次数少（均摊 O(1)），而查找效率略低于 AVL。在内存中读多写少的场景 AVL 可能更快，但红黑树更通用。不要用平衡高度或查找复杂度来评判二者，而应关注重平衡成本。

思考题：在删除修复过程中，如果父节点被旋转后，原双重黑节点 x 的兄弟 w 的父亲发生了变化，为什么情形4可以直接把 x 置为根终止，而不会破坏性质5？请尝试通过黑高的局部守恒证明：在执行情形4前后，从任意节点到叶子路径的黑高保持全局不变，从而说明终止条件的正确性。
