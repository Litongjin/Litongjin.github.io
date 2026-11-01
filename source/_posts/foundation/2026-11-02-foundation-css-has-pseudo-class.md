---
title: "每日基础技术总结 · 2026-11-02 · CSS :has() 伪类选择器：相对后代选择的历史突破与匹配算法复杂性"
date: 2026-11-02 07:01:51
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-02 · CSS :has() 伪类选择器：相对后代选择的历史突破与匹配算法复杂性

## 📚 今日主题

> **CSS :has() 伪类选择器：相对后代选择的历史突破与匹配算法复杂性**（前端底层与计算机基础）

### 1. 核心概念速览
**定义**：`:has()` 是 Selectors Level 4 定义的**关系伪类（relational pseudo-class）**，函数式，参数为 `<relative-selector-list>`。形式语义：元素 E 匹配 `E:has(rs)` 当且仅当存在元素 F，使得以 E 为**锚点（anchor）**对 rs 求值时 F 匹配，即 ∃F : match(F, anchor(E, rs))。

**本质**：CSS2.1 以来，选择器的「主体（subject）」恒为最右侧复合选择器，匹配引擎据此固定为右到左回溯。`:has()` 把主体留在左侧，将参数内的相对选择器显式锚定在该主体上，使匹配方向翻转为「从锚点沿指定组合符轴做存在量化」。它没有引入新语法范畴，而是把「反向关系查询」封装进函数参数，从而保持了「最右复合选择器即主体」这一引擎不变量。

**解决的问题**：无法基于后代状态选择祖先（「父选择器」的诉求），以及无法基于后继兄弟状态选择前置兄弟。历史上 `:contains()`、subject indicator（`!div > p` / `div! > p`）均因语法歧义、破坏主体不变量、表达能力不足（无法表达兄弟轴）而被废弃；`:has()` 是其等价且更强、且不污染选择器语法的替代。

**体系位置**：CSS 选择器引擎 → 样式系统（Style Resolution / RecalcStyle）→ 样式失效（Style Invalidation）。它横跨三个子系统：解析期（语法与特异性）、匹配期（树上的存在量化）、失效期（DOM 变更后的增量重算）。在渲染流水线中，它的成本主要不落在「匹配」，而落在「变更传播」。

**为什么必须掌握**：`*:has(...)` 本质是「树上带存在量化的模式查询 + 增量视图维护（incremental view maintenance）」，这与数据库物化视图刷新、响应式系统的依赖追踪是同一问题族的极小样本。掌握它，等于同时掌握选择器求值方向、特异性的组合规则，以及浏览器失效传播的成本模型——这是判断「某条 CSS 是语法问题、匹配问题，还是失效抖动问题」的分水岭。

### 2. 底层原理剖析
## 1. 语法与形式语义

    <complex-real-selector> ::= <compound> ( <combinator> <compound> )*
    <relative-selector>     ::= <combinator>? <complex-real-selector>
    <combinator>            ::= '>' | '+' | '~' | ' '      // 省略即后代
    :has( <relative-selector-list> )

锚点集合 Anchor(E, rs) 由 rs 的前导组合符唯一决定：

    ' '  -> descendants(E)
    '>'  -> children(E)
    '+'  -> nextSibling(E)
    '~'  -> followingSiblings(E)

关键：相对选择器**没有祖先组合符**，因此 `:has()` 无法表达「存在某个祖先满足 S」；同时 `+`/`~` 使匹配目标可以落在锚点子树**之外**，这直接决定了失效传播的边界。

## 2. 特异性

`:has()` 对特异性的贡献 = 参数列表中**单条复杂选择器特异性的最大值**（与 `:is()`/`:not()` 同规则），不是求和，组合符本身不计分。

    li:has(> input:checked)  ->  li(0,0,1) + max{(0,1,1)} = (0,1,2)
    p:has(.a, #k)            ->  p(0,0,1)  + max{(0,1,0),(1,0,0)} = (1,0,1)

## 3. 语法约束

- **非宽容（non-forgiving）**：参数列表中任意一项无效，整个 `:has()` 无效，整条规则被丢弃。规范在 2022 年由 forgiving 改为 non-forgiving，以对齐 jQuery 的 `:has()` 并避免「静默降级导致语义漂移」。对比 `:is()`/`:where()` 是宽容的（丢弃无效项、保留其余）。
- 参数内禁止伪元素；`:has()` 内部不可再嵌套 `:has()`。
- `:has()` 只能作用于元素，不能挂在伪元素上。

## 4. 匹配算法：右到左 + 局部前向量化

标准匹配自右向左。含 `:has()` 时，锚点 E 本身就是最右复合选择器，命中后对参数做前向存在量化，且必须短路：

    function matchesComplex(E, sel):
        if !matchCompound(E, sel.rightmost): return false
        if sel.rightmost has :has(rsList):
            if !matchHas(E, rsList): return false
        return matchLeftward(E, sel.rest)       // 常规右到左回溯祖先

    function matchHas(E, rsList):
        for rs in rsList:
            for F in axisOf(rs.leadingCombinator, E):
                if matchRightToLeft(F, rs.body, stopAt = E):
                    return true                  // 存在量化：一命中即返回
        return false

朴素的「每次样式重算都全量跑上述算法」在 N 个节点上最坏是 O(N·|subtree|)，`*:has(* *)` 逼近 O(N²)。因此生产引擎采用的是**失效驱动 + 静态剪枝 + 结果缓存**，而非重算：

    onMutation(node N, change):
        for rule in styleRules where rule.containsHasPseudo:
            // (a) 祖先传播：N 的任意祖先都可能成为 :has() 的锚点
            for A in N.ancestors():
                if fastReject(A, rule.descendantBloomFilter): continue
                markStyleDirty(A)

            // (b) 兄弟轴：rs 以 '+' / '~' 前导时，锚点可能在前驱兄弟上
            if rule.hasSiblingRelativeSelector:
                for S in N.precedingSiblingsAndTheirAncestors():
                    markStyleDirty(S)

            // (c) 动态状态：:hover/:focus/:checked 等位变化无法静态剪枝
            if rule.containsDynamicState:
                propagateAncestorsAffectedByHas(N)

工程实现要点（以 Blink 为例）：`SelectorFilter` 为每个锚点维护后代 class/id/tag 的 Bloom Filter（descendant filter），`FastRejectSelector` 在向子树下降前先判定「该子树里根本不可能出现参数中的复合选择器」，把绝大多数候选剪掉；`:has()` 的匹配结果按 `(selector, anchor)` 进 `RelationCache`，失效时精确驱逐；而锚点元素会被打上 `AncestorsAffectedByHas` 标记，使后续变更能快速定位受影响的锚点集。

## 5. 复杂度结论

- 静态 `:has(> .x)`：Bloom Filter 可拒绝大量候选 + RelationCache 命中，摊销后近似线性。
- 动态 `:has(:hover)`：状态位无法用静态过滤器剪枝，任何后代的状态翻转都要向祖先广播，成本与「鼠标移动频率 × 祖先链长度」成正比。
- 危险形态是「**高扇出锚点 × 动态伪类**」：`body:has(.x:hover)` 会让每次 mousemove 触发整条祖先链的候选失效。

## 6. 与前端已有概念的异同

- `:is()`/`:not()`/`:where()`：参数是复杂选择器列表，但**不跨越组合符**，只在同一主体的匹配路径上做布尔运算；特异性规则与 `:has()` 相同。
- `:focus-within`：引擎内建的、针对特定状态的特化反向伪类；`:has(:focus)` 是它的泛化，但前者有专用的状态位与更廉价的失效通道。
- `el.querySelector(sel) !== null`：命令式一次性求值，结果不随 DOM 变化自动更新；`:has()` 是声明式，参与引擎失效传播，状态由内核维护。
- `Element.closest()`：沿祖先轴向上，方向与 `:has()` 相反且不可互换——`E:has(F)` ≠ `F:has(E)`。
- `:nth-child(An+B of S)`：同样是对「其他元素集合」求值并按 S 过滤，但作用于兄弟集合，特异性规则亦为取最大值。

### 3. 基础代码与实战验证
```text
以下为无框架、可直接运行的最小验证页，覆盖形式语义、失效传播、语法约束、特异性四条原理。

    <style>
      /* [1] 子轴：锚点是 li，先匹配 li 复合选择器，再前向检查 children 轴 */
      li:has(> input:checked) { background: #d7f5d7; }

      /* [2] 兄弟轴：锚点是 h2；'+' 使匹配目标落在 h2 的兄弟，位于其子树之外 */
      h2:has(+ p) { margin-bottom: 0; }

      /* [3] 特异性取参数中最高者而非求和：
             p:has(.a, #k)  -> (1,0,1)（id 单列胜出）
             p:has(.a.b.c)  -> (0,3,1)
             若误认为 :has() 不计特异性，则蓝色会胜出 */
      p:has(.a, #k) { color: red; }
      p:has(.a.b.c) { color: blue; }

      /* [4] 非宽容：只要有一项无效，整条规则被丢弃（对比 :is() 会丢弃无效项保留其余） */
      /* p:has(.x, :bogus) { color: green; } */
    </style>

    <ul id="list">
      <li><input type="checkbox">A</li>
      <li><input type="checkbox">B</li>
    </ul>
    <p id="k" class="a b c">probe</p>

    <script>
      const li  = document.querySelector('#list li');
      const box = li.querySelector('input');

      // A. 存在量化的直接求值：matches() 以 li 为锚点跑 matchHas()
      console.log(li.matches(':has(> input)'));         // true  —— children 轴存在 input
      console.log(li.matches(':has(> input:checked)')); // false —— 初始 checkedness 为 false

      // B. 失效传播验证：全程不写任何 class 操作，只改表单状态位;
      //    引擎在属性/状态变更后沿祖先链标记 :has() 锚点脏，并精确重算 li 的声明
      box.checked = true;
      console.log(getComputedStyle(li).backgroundColor); // rgb(215, 245, 215)
      // 注意：样式翻转由内核 invalidation 驱动，主线程未参与选择器求值；
      // getComputedStyle 只是强制同步一次 style recalc，不触发我们的 JS 逻辑

      // C. 语法约束验证
      try { document.querySelector('#list li:has(> input, :bogus)'); }
      catch (e) { console.log(e.name); }  // SyntaxError —— :has() 非宽容，整条无效

      try { document.querySelector('#list li:has(::before)'); }
      catch (e) { console.log(e.name); }  // SyntaxError —— 参数内禁止伪元素

      // D. 特异性验证：(1,0,1) vs (0,3,1)，id 列优先 -> red
      console.log(getComputedStyle(document.querySelector('#k')).color); // rgb(255, 0, 0)

      // E. 复杂度对比（在同一大 DOM 上取样）
      //    ':has(.leaf)' 每次 querySelectorAll 都全量前向量化；
      //    而静态子轴 :has(> .leaf) 可被 descendant Bloom Filter 快速拒绝，
      //    两者在深树 + 稀疏匹配场景下的耗时差可达数量级。
    </script>
```

### 4. 常见误区与进阶思考
## 误区一：把 `:has()` 当「父选择器」

`:has()` 是对**相对选择器**的存在量化，语义由前导组合符决定，默认是后代。`h2:has(+ p)` 匹配的目标完全不在 h2 的子树内；`:has(~ .x)` 甚至指向 h2 之后的所有兄弟。更关键的是，相对选择器语法里**不存在祖先组合符**，所以 `:has()` 根本无法表达「存在某个祖先满足 S」——那属于 `:is()` 配合后代选择器，或 JS 的 `closest()`。同理 `E:has(F)` 与 `F:has(E)` 不可互换：前者的主体是 E、量化方向向前，后者相反。把它理解成「反向后代选择器」会在兄弟轴和特异性上连续出错。

## 误区二：认为 `:has()` 的性能问题出在「选择器匹配」

真正的成本在**失效传播**。静态 `:has(> .x)` 有 Bloom Filter 静态剪枝和 `(selector, anchor)` 级缓存，摊销后可接受；一旦参数里出现动态伪类（`:hover`、`:focus`、`:focus-within`、`:checked`），状态位的变化无法用静态过滤器排除，必须把脏标记向祖先链广播，且兄弟轴（`+`/`~`）还要求把失效集扩展到锚点的**前驱兄弟**——这比单纯的祖先传播更难剪枝。因此 `body:has(.item:hover)`、`:root:has(dialog[open])` 这类高扇出锚点，是典型的高频抖动来源，与「选择器写得长不长」毫无关系。

另有一个次级误区：把 `:has()` 的宽容性与 `:is()` 混同。`:has()` 是非宽容的，一个拼错的伪类会让整条规则消失，且不会报错到控制台——排查时容易被误判为「样式没生效」而非「规则被丢弃」。

## 思考题

设有选择器 `A:has(~ B C)`：A 与 B 是兄弟，C 是 B 的后代。现在在 C 的子树深处插入一个匹配 C 复合选择器的节点。该节点的祖先链覆盖 C、…、B、其父节点，但 **A 只是 B 的前驱兄弟，不在这条祖先链上**。

请回答：
1. 引擎必须如何扩展失效集，才能保证 A 的样式被重算？为什么这个扩展方向（沿兄弟轴做「反向」传播）比纯祖先传播更难做剪枝？
2. 据此判断，在动态 DOM（频繁增删节点）场景下，`:has(~ ...)` 与 `:has(> ...)` 的成本差异体现在哪一层：匹配层、失效传播层，还是样式重算层？
3. 若把选择器改写为 `A:has(.x)` 且保证 `.x` 不会动态变化，成本模型会如何改变？
