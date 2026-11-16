---
title: "每日基础技术总结 · 2026-11-17 · JavaScript Set 与 Map 的底层哈希表实现：碰撞解决与迭代顺序保证"
date: 2026-11-17 07:02:36
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-17 · JavaScript Set 与 Map 的底层哈希表实现：碰撞解决与迭代顺序保证

## 📚 今日主题

> **JavaScript Set 与 Map 的底层哈希表实现：碰撞解决与迭代顺序保证**（前端底层与计算机基础）

### 1. 核心概念速览
Map/Set 是 ECMAScript 规范化的关联容器。Set 是基于 SameValueZero 等价的不可重复有序集合；Map 是 key->value 有序关联表，key 可为任意 JS 值。其解决的是 Object 键只限 string/symbol、缺少 size、原型键污染、键类型与插入语义不够严格的问题。语言层规定操作语义和迭代顺序，不规定物理实现；工程实践通常以哈希表作为索引容器以提供平均 O(1) 查找，再叠加顺序容器满足规范。在缓存、路由、状态索引、推荐系统特征访问、AI 推理侧样本索引等后端与数据路径中，它们是高频原语；专业工程师需要区分规范语义、引擎实现策略与性能退化点。

### 2. 底层原理剖析
一、键规范化与相等性：Map/Set 使用 SameValueZero。Set 的 primitive key 按类型与值判定，1 与 '1' 不同；NaN 与 NaN 视为同一 key；+0 与 -0 视为同一 key，且 Map/Set 内部通常将 -0 归一为 +0。Object key 使用引用身份，不按结构深度相等。二、哈希索引与碰撞解决：key 经引擎内部 hash 生成桶下标；桶内碰撞可通过链地址、开放寻址、二次探测、Robin Hood 等策略解决，具体由引擎选择。同桶多 key 查找仍需逐个做 SameValueZero 比较。负载因子超过阈值后 rehash 会改变桶分布，但不改变逻辑键值与迭代顺序。三、迭代顺序保证：纯哈希桶按桶下标或内存布局遍历不保证插入顺序；JS 规范强制 Map/Set 按 key 插入顺序迭代。因此有序哈希容器必须把物理桶索引与逻辑顺序分离。常见实现是 entry list/order array/linked list：新建 key 时追加 entry 到顺序结构；更新已有 key 时只替换 value，不移动 entry；删除 key 时从顺序结构摘除或打 tombstone 后压缩；迭代时遍历顺序结构而非桶数组。四、基础流程伪代码：INSERT(k,v): normalizedKey = normalize(k); bucket = table[hash(normalizedKey)]; existing = findByEquality(bucketChainOrProbe, normalizedKey); if existing exists: existing.value = v; return; entry = create(normalizedKey, v); appendToOrder(entry); insertIntoBucket(bucket, entry, collisionPolicy). GET(k): normalizedKey = normalize(k); probe bucket chain/open addressing; compare by SameValueZero; return entry.value or undefined. DELETE(k): found = probe; remove from order list or mark tombstone; decrement size. ITERATE(): walk order structure head to tail; yield entry.value.五、复杂度与代价：平均 O(1) 插入、删除、查找；O(n) 遍历；哈希退化时可趋近 O(n)。额外维护顺序结构带来内存与指针/索引间接层，但这是满足规范顺序语义的必要成本。六、与前端已有概念对比：Object 的键受 string/symbol 限制，整数型键有独立排序规则，不适合作任意值关联容器；Java HashMap 迭代顺序不规定，LinkedHashMap 才提供插入顺序；JS Map/Set 在语言规范层面直接规定插入顺序，但允许引擎用不同哈希冲突策略实现；TS 只描述运行时 API 类型，不改变运行时哈希表、SameValueZero 或顺序保证。

### 3. 基础代码与实战验证
```text
const map = new Map()
// 首次 set: 归一化 key -> 计算桶下标 -> 按冲突策略定位 -> 新建 entry -> 追加到有序 entry 链或数组
map.set('a', 1)
map.set('b', 2)
// 更新已有 key: SameValueZero 命中同一 entry，只替换 value，不移动逻辑插入顺序
map.set('a', 11)
console.log([...map.keys()]) // 规范保证按插入顺序: a, b

map.delete('a')
map.set('a', 22)
console.log([...map.keys()]) // 删除后再插入到末尾: b, a

// Set 使用 SameValueZero: 数字 1 与字符串 '1' 不等价；-0 与 +0 等价
const set = new Set([1, '1', -0])
console.log([...set]) // 1, '1'

// Map/Set 的 key 类型无关；对象 key 使用引用身份，不是结构相等
const o1 = { x: 1 }
const o2 = { x: 1 }
const objectMap = new Map()
objectMap.set(o1, 'o1')
console.log(objectMap.get(o1), objectMap.get(o2)) // o1, undefined

// NaN 作为 Map key 可命中同一 entry；这与严格相等 NaN !== NaN 不同
const nanMap = new Map()
nanMap.set(NaN, 'nan')
console.log(nanMap.get(NaN)) // nan
```

### 4. 常见误区与进阶思考
误区一：把 Map/Set 当作纯哈希表，认为迭代顺序取决于桶顺序或碰撞链顺序。错误点在于规范保证插入顺序；引擎必须先满足顺序语义，再让哈希表只作为索引容器。碰撞解决方式改变、rehash、桶扩容都不应改变可见迭代顺序。误区二：把 SameValueZero 混同为 === 或 Object.is。=== 下 NaN !== NaN，且 +0 === -0；Object.is 下 NaN 相同但 +0 与 -0 不同；Map/Set 采用 SameValueZero，因此 NaN 可作为同一 key，-0 与 +0 可作为同一 key。该差异会影响缓存键、数值零边界、ID 生成与特征索引正确性。思考题：如果用开放寻址哈希表实现插入顺序 Map，删除时只标记 tombstone，扩容后 tombstone 与 entry 下标都会变化，如何在不扫描完整桶数组的前提下同时保证平均 O(1) 插入、O(n) 有序遍历、内存回收和 GC 引用可达性？
