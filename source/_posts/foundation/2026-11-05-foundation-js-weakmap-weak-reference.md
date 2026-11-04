---
title: "每日基础技术总结 · 2026-11-05 · JavaScript WeakMap 与弱引用：内存泄漏防护与服务关联数据的存储"
date: 2026-11-05 07:02:09
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-05 · JavaScript WeakMap 与弱引用：内存泄漏防护与服务关联数据的存储

## 📚 今日主题

> **JavaScript WeakMap 与弱引用：内存泄漏防护与服务关联数据的存储**（前端底层与计算机基础）

### 1. 核心概念速览
**定义**：WeakMap 是以对象（或非注册 Symbol）为键、键为**弱引用（weak reference）**的键值表。规范层面它持有内部槽 [[WeakMapData]]，其条目对键不构成强引用：登记一条 weakMap.set(k, v) 既不延长 k 的存活期，也不会因为该条目让 k 成为可达对象。值默认是强引用，但整个条目受 **ephemeron（条件强引用）** 语义约束——v 是否存活取决于 k 是否存活。

**它解决的问题**：需要以「对象身份」为索引挂载附属数据（元数据、缓存、订阅关系、私有状态），又不希望这条登记关系成为对象无法回收的原因。Object 属性、Map、闭包捕获都会把附属数据与被附着对象绑定成同一生命周期；在长驻进程中，这只是把生命周期从业务代码转移到了 GC 根，最终造成渐进式内存泄漏。

**机制本质**：GC 的可达性分析中，常规强边会被传播标记（mark）；WeakMap 条目不是从 key 出发的强边，而是由 mark 阶段末尾对 ephemeron 表做**不动点迭代**来处理：只有当 key 已被证明存活时，才把 value 传播为存活；key 未被标记的条目会在同一次 GC 内被原子删除（key 与 value 若仅被该条目引用，一并回收）。

**在计算机体系中的位置**：横跨「运行时内存管理（GC 弱边、ephemeron 不动点算法）」与「数据结构（身份哈希表）」两层。它是所有以对象身份为键的关联存储的基础设施：Vue 3 的 targetMap 依赖收集、DOM 节点元数据表、Node/SSR 常驻进程中的请求上下文绑定、AI 推理服务里 session→张量缓存的键控生命周期。专业工程师必须掌握，因为判断「一个缓存/注册表是否会泄漏」的第一性依据就是可达性图上的边强度，而长驻进程（Node 网关、SSR 服务、常驻推理服务）的稳定性事故绝大多数源于不可回收的强引用链。

### 2. 底层原理剖析
**1. 引用强度与可达性三态**
GC roots（栈、全局对象、活跃闭包、寄存器）出发做图遍历。对象状态可分为：强可达（存在全强边路径）、弱可达（只经由弱边/ephemeron 条目到达）、不可达。后两类在本次 GC 中都会被回收，但弱可达对象在回收前会触发 WeakRef 失效与 FinalizationRegistry 登记清理。

**2. 朴素「弱 key + 强 value」为什么不够：环形保活**
若实现为 table: { weakKey -> strongValue }，且 value 反向引用 key（极常见：{ owner: key }、缓存条目回指、闭包捕获），则：table 本身强可达 → value 强可达 → value 强引用 key → key 强可达 → key 永不被回收，条目永不被清除。这正是 Java `WeakHashMap` 的已知缺陷（其 value 由 map 强持有），必须配合 ReferenceQueue 手动清理，或改用 `WeakReference` 包 value。

**3. Ephemeron 语义（不动点迭代）**
条目整体是「条件强引用」：value 的强度 = key 的强度。伪代码：

```
mark_stack = scan_roots()                  // 常规强引用传播
repeat:
    for entry in ephemeron_table:          // 迭代直到不再产生新标记
        if is_marked(entry.key) and not is_marked(entry.value):
            mark(entry.value); changed = true
until not changed
for entry in ephemeron_table:
    if not is_marked(entry.key):
        delete entry                       // 原子：key 不可达 ⇒ 条目与 value 同时不可达
```

必须迭代到不动点，因为存在链式 ephemeron（wm1: a→b, wm2: b→c），单趟扫描会漏标。第 2 节的环在 ephemeron 下自然消失：key 只被 value 引用，value 只在 key 已标记时才被标记 —— 无种子，两者皆不可达，同一次 GC 清除。

**4. 引擎实现要点（以 V8 为例）**
WeakMap 内部表是独立类型 `EphemeronHashTable`（与 NameDictionary/NumberDictionary 并列），条目哈希基于对象的 **identity hash**（惰性生成并缓存于对象头/属性后备存储，因此把一个对象首次作为 WeakMap key 需付出一次哈希计算开销）。mark-compact 中，弱引用与 ephemeron 在「weak closure」阶段统一处理，清除与回收发生在同一次 GC 内，不对外暴露「key 已死但条目仍在」的中间态。

**5. 由语义反推 API 契约**
- 只有 get/set/has/delete，**没有 size、keys、values、forEach、clear**：暴露条目数或枚举就等于把 GC 时机变成语言可观测状态，程序语义将依赖具体引擎策略；`delete` 只需单条目寻址，不违反该约束。
- 不可枚举 ⇒ 不可结构化克隆、不可序列化、不能作为 JSON 通道的数据载体。
- 键必须是 Object 或非注册 Symbol：原始值（number/string）无身份与生命周期概念；`Symbol.for(x)` 注册在全局符号表中永久可达，作为弱键毫无意义，规范直接排除（非注册 Symbol 已在 ES2024 起允许）。

**6. WeakRef 与 FinalizationRegistry 配套**
- `new WeakRef(t).deref()` 返回 t 或 undefined。规范 `KeepDuringJob`：一旦在某个 job（同步执行块）内 deref 出 t，t 保证在该 job 结束前不被回收，防止同一段同步代码内的 TOCTOU 失效。跨 `await` 后不保证。
- `FinalizationRegistry.register(target, heldValue)`：target 弱持有，heldValue 强持有（因此 heldValue 不得引用 target）；回调在 GC 之后由宿主异步调度，规范允许其永不执行。

**7. 与既有概念的对照**

| 机制 | 键 | 值 | 可枚举 | 生命周期耦合 |
|---|---|---|---|---|
| Object 属性 / Map | 强 | 强 | 是 | 完全耦合 |
| 闭包捕获 | 强 | 强 | — | 隐式耦合 |
| WeakMap | 弱（ephemeron） | 条件强 | 否 | 单向解耦 |
| Java WeakHashMap | 弱 | 强（非 ephemeron，需 ReferenceQueue） | 是 | 半解耦，有环泄漏风险 |
| C++ weak_ptr | 弱（控制块引用计数） | — | — | 可 lock() 升强，WeakMap 无此语义 |
| .NET ConditionalWeakTable | 弱（ephemeron） | 条件强 | 可枚举 | 与 WeakMap 语义最接近 |

### 3. 基础代码与实战验证
```text
// 文件 weakmap-demo.mjs   运行 node --expose-gc weakmap-demo.mjs
// --expose-gc 暴露 global.gc()，仅用于观测「生产代码不可观测的 GC 时机」

const finalized = [];
const reg = new FinalizationRegistry(tag => finalized.push(tag));
// register(target, heldValue)：target 弱持有；heldValue（此处 tag）强持有，
// 因此 heldValue 绝不能引用 target —— 否则等于亲手建立一条指向 target 的强路径

// ---------- 1. 弱边：登记关系不延长 key 的存活期 ----------
const wm = new WeakMap();
let key = { id: 1 };      // let 绑定持有强引用，对象此刻强可达
reg.register(key, 'k1');
wm.set(key, 'meta');      // wm -> key 是弱边，不参与标记传播
key = null;               // 断开唯一显式强引用，对象转为弱可达
console.log('before gc, wm entry exists =', wm.has({})); // false：无法用新对象探测
// 关键事实：没有任何活着的引用可以再查询该条目 —— 弱映射的可观测性只存在于 key 存活期间
global.gc();                                          // 触发一次 mark-compact
await new Promise(r => setTimeout(r, 0));             // FR 回调在 GC 栈之外异步调度，必须让出事件循环
console.log('after gc, finalized =', finalized);      // ['k1']：条目已随 key 一并清除

// ---------- 2. 强边对照：Map 会让 key 永不回收 ----------
const strong = new Map();
let k2 = { id: 2 };
reg.register(k2, 'k2');
strong.set(k2, 'meta');   // strong（自身强可达）-> k2 是强边，k2 被保活
k2 = null;
global.gc();
await new Promise(r => setTimeout(r, 0));
console.log('strong.size =', strong.size, 'finalized =', finalized);
// strong.size 仍为 1 且 finalized 中无 'k2'：key 不可达性被 Map 内部槽彻底否决

// ---------- 3. Ephemeron：value 反向引用 key 不构成泄漏 ----------
const eph = new WeakMap();
let node = {};
reg.register(node, 'node');
// 条目内成环：eph 弱引 node，而 value 强引用 node
// 若实现为「弱 key + 强 value」（如 Java WeakHashMap），该环使 node 永远强可达；
// ephemeron 下：node 未被标记 ⇒ 条目条件不成立 ⇒ value 不被标记 ⇒ 两者同批回收
eph.set(node, { owner: node });
node = null;
global.gc();
await new Promise(r => setTimeout(r, 0));
console.log('cycle collected =', finalized.includes('node')); // true

// ---------- 4. 关联元数据：不污染对象、不延长生命周期 ----------
const privateState = new WeakMap();   // 等价于 #private 字段出现前的标准品牌校验方案
class Session {
  constructor(token) { privateState.set(this, { token, createdAt: Date.now() }); }
  get token() {
    const s = privateState.get(this);   // 非 Session 实例返回 undefined ⇒ 天然的品牌检查
    if (!s) throw new TypeError('incompatible receiver');
    return s.token;
  }
}

// ---------- 5. 两级索引：外层弱、内层必须强 ----------
const targetMap = new WeakMap();      // 外层弱：target 被回收 ⇒ 整棵依赖子树随之不可达（无需手动清理）
let currentEffect = null;
function track(target, propKey) {
  let depsMap = targetMap.get(target);
  if (!depsMap) targetMap.set(target, (depsMap = new Map()));
  // 内层必须是 Map：propKey 是字符串/Symbol 原始值，无身份与生命周期，
  // 用 WeakMap 既语义错误也会直接抛 TypeError
  let dep = depsMap.get(propKey);
  if (!dep) depsMap.set(propKey, (dep = new Set()));
  dep.add(currentEffect);             // 对 effect 是强引用：依赖的清理责任在调度器/卸载钩子，不在 WeakMap
}
```

### 4. 常见误区与进阶思考
**误区一：把 WeakMap 当作「不会泄漏的缓存」**
换成 WeakMap 并不改变 key 到 GC root 的强路径，它只是一个旁观者。若 key 仍被闭包、全局注册表、事件监听器、长生命周期 Map 或 pending Promise 强持有，条目照样永不回收。更关键的是：WeakMap 无 size、不可枚举、无 clear，因此无法承载任何需要容量统计与淘汰策略的缓存（LRU/LFU/TTL/SWR）。它真正的语义是「生命周期完全从属于宿主对象的附属数据表」，而非缓存。需要可淘汰缓存时，必须显式组合 `Map<string, WeakRef>` + `FinalizationRegistry` 异步回收（同样只能做补救，不能保证及时性），或引入显式生命周期管理。

**误区二：认为 WeakRef/FinalizationRegistry 可用于资源释放**
`deref()` 的 KeepDuringJob 只覆盖当前同步 job，跨 `await` 之后 target 可能已失效，不能把 WeakRef 当「弱句柄 + 异步安全」使用；`FinalizationRegistry` 回调执行时机不确定、可能被推迟到进程退出、规范明确允许永不执行。因此它只能用于清理「可选的内存型附属数据」（日志、统计、缓存条目、调试信息），绝不能用于释放 fd、文件锁、数据库连接、租约等必须确定性释放的资源 —— 后者只能依赖显式 close/dispose 协议（try/finally、Symbol.dispose、作用域钩子）。另一个同源误区是认为 value 引用 key 一定造成泄漏，从而人为拆环、引入额外弱层，实际上 ephemeron 已处理该环。

**思考题**：WeakMap 提供了 `delete` 却刻意不提供 `size` 与 `forEach`。请从「并发/增量标记」角度论证：若要为 `size` 提供一个线性一致的值，引擎需要在 mutator 读操作与标记线程之间建立什么同步？为什么这种同步会把「GC 何时运行、采用何种策略」泄漏为语言层面可观测的状态，进而使程序语义与具体引擎实现绑定？并进一步思考：Vue 3 之所以用 `WeakMap<target, Map<key, Set<effect>>>` 而非三层 WeakMap，其内层 key 是原始值这一事实与上述 API 契约之间是否存在同构的设计约束？
