---
title: "每日基础技术总结 · 2026-11-10 · JavaScript Module System：ESM 的静态分析与动态导入（Import()）执行流"
date: 2026-11-10 07:03:20
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-11-10 · JavaScript Module System：ESM 的静态分析与动态导入（Import()）执行流

## 📚 今日主题

> **JavaScript Module System：ESM 的静态分析与动态导入（Import()）执行流**（前端底层与计算机基础）

### 1. 核心概念速览
ESM 是 ECMAScript 语言级的模块系统（规范 §16.2 Modules），本质是一套「解析期静态确定的模块图 + Link/Evaluate 两阶段执行模型」。

它解决的问题：在语言层给出可静态分析的依赖与导出集合，使 Host（浏览器 / Node / 打包器）能在**不求值任何模块代码**的前提下完成依赖解析、绑定建立与图完整性校验。

机制要点：
1. Parse 阶段（ParseModule）从 AST 中抽出 ImportDeclaration / ExportDeclaration，产出模块记录的四张静态表：[[ImportEntries]]、[[LocalExportEntries]]、[[IndirectExportEntries]]、[[StarExportEntries]]，以及 [[RequestedModules]]。这一步不执行任何代码，所以 specifier 必须是 StringLiteral，import 必须位于顶层。
2. Link 阶段（Instantiate）按 DFS 递归解析 specifier（HostResolveImportedModule），为每个模块创建 Module Environment Record，把所有 import 绑定建成**指向目标模块 binding 的间接绑定**——此时只建立引用关系，不读取值。
3. Evaluate 阶段按 DFS 后序执行模块体（[[ECMAScriptCode]]）。

与 import() 的分界：静态 import 的 resolve / link / evaluate 全部在宿主加载模块图时一次性、深度优先地完成；import() 是一个**运行期求值的 CallExpression**，把「解析 specifier、加载、link、evaluate」推迟到调用点，通过宿主钩子 HostImportModuleDynamically 异步完成并返回 Promise。二者共享同一 Realm 的 Module Map，因而命名空间对象与副作用执行次数仍是一次。

在体系中的位置：它是现代前端构建链路的地基——打包器的依赖图、tree-shaking 的可行性、code-splitting 的 chunk 边界、Node 对 CJS/ESM 互操作的约束（require(esm) 与 TLA 的冲突）、浏览器 <script type="module"> 的 defer 语义，全部由这套静态结构决定。前端工程师若只把它当「换一种 import 写法」，就无法解释为什么某些导出摇不掉、为什么循环依赖有时是 undefined 有时是 ReferenceError、为什么同一个模块被两次 import() 只执行一次。

### 2. 底层原理剖析
一、Module Record 的静态产物（为什么叫「静态」）

Module Record 在 ParseModule 后即持有完整依赖信息，与运行期状态无关：

  [[RequestedModules]] : List<ModuleSpecifier>          // 全部 import/export-from/import() 之外的字面量依赖
  [[ImportEntries]]    : { ModuleRequest, ImportName, LocalName }[]
  [[LocalExportEntries]] / [[IndirectExportEntries]] / [[StarExportEntries]]
  [[Status]]           : unlinked | linking | linked | evaluating | evaluating-async | evaluated
  [[EvaluationError]]  : 求值失败后缓存的错误（不再重试）
  [[DFSIndex]] / [[DFSAncestorIndex]]  // 用于识别强连通分量（SCC）

这四张表是「静态分析」的全部物理依据。bundler 能构建确定的依赖图、能做 tree-shaking、能判定 `export * from` 的同名冲突（ResolveExport 返回 ambiguous），都是因为该表在求值前已经存在；CommonJS 的 exports 是运行期可变对象，依赖只能在执行中逐步发现，无法获得同等保证。

二、Link（实例化）：只建绑定，不取值

  InnerModuleLinking(module, stack, index):
    if module.[[Status]] in {linking, linked, evaluated}: return index   // 环检测：命中即返回
    module.[[Status]] = linking
    module.[[DFSIndex]] = module.[[DFSAncestorIndex]] = index++
    for req of module.[[RequestedModules]]:
        required = HostResolveImportedModule(module, req)   // 宿主负责 URL 解析 + 加载 + 递归 Parse
        index = InnerModuleLinking(required, stack, index)
        if required.[[Status]] == linking:
            module.[[DFSAncestorIndex]] = min(module.[[DFSAncestorIndex]], required.[[DFSAncestorIndex]])
    module.[[Status]] = linked
    stack.push(module)                       // 后序：栈顶为最深层依赖

关键点：Link 阶段创建 Module Environment Record 并为每个 import 建立**间接绑定（indirect binding）**，指向被导入模块环境槽位；被导入的 let/const 槽位此刻仍是未初始化态（TDZ）。因此「绑定已存在」与「值未就绪」可以同时成立——这正是 ESM 循环依赖能运转、但同步读取仍会抛 ReferenceError 的根因。

三、Evaluate：DFS 后序 + SCC

  InnerModuleEvaluation(module):
    if module.[[Status]] == evaluated: return module.[[EvaluationError]]
    if module.[[Status]] == evaluating:
        return index                    // 环：不重复入栈，由 SCC 根统一调度
    module.[[Status]] = evaluating
    for req of module.[[RequestedModules]]:
        InnerModuleEvaluation(resolve(req))
    ExecuteModule(module)               // 执行 [[ECMAScriptCode]]
    module.[[Status]] = evaluated

真正的执行顺序是「依赖 → 依赖者」的后序，同一 SCC 内按 DFS 完成序执行。ESM 相对 CJS 的改进只是把「绑定声明」提前到了 Link（hoisting 语义），执行顺序依旧是依赖者可能在依赖完成前被调用，所以 `const` 仍处 TDZ，而 `function` 声明在 Environment Record 实例化时已初始化（可用）。也可对照 Java 的 import：它仅是编译期符号可见性声明，不进入运行期依赖图；TS 的 interface 与 import type 在类型擦除后不留任何运行时痕迹，而 ESM 的 import 会在 Link 阶段生成真实绑定对象——三者不是一个层面的机制。

四、import() 的执行流

  EvaluateCall(import, [specifier]):
    p = NewPromiseCapability(%Promise%)
    HostImportModuleDynamically(referencingScriptOrModule, specifier, p)
    return p

宿主的典型实现（浏览器 / Node）：
1. 对 specifier 求值（可为任意表达式，不必是字面量）；
2. 以 referrer 的 base URL 解析为绝对 URL；
3. 查 Module Map[key]：命中则直接拿已有 Module Record，否则 fetch + ParseModule；
4. 在新 Record 上执行 InnerModuleLinking + InnerModuleEvaluation（整体作为异步任务），
5. Evaluate 成功后以 [[Namespace]]（Module Namespace Exotic Object）resolve Promise；失败则 reject 并写入 [[EvaluationError]]，之后任何 import() 同一 URL 都直接 reject 同一错误，不重新求值。

三点与静态 import 的实质差异：时机（运行期 vs 加载期）、图结构（不进 [[RequestedModules]]，无法被静态分析，只能成为 chunk 边界）、错误传播（Promise reject vs 整个模块图 evaluate 失败、[[EvaluationError]] 沿依赖链向上冒泡）。而「单例语义」不变：同一 Realm 的 Module Map 保证副作用只执行一次，多个 import() 得到的是同一个命名空间对象的同一个引用（但 Promise 实例不同）。若目标模块含顶层 await，该 Record 的 [[AsyncEvaluation]] 为真、[[PendingAsyncDependencyCount]] 大于零，Promise 的 resolve 被推迟到异步依赖全部完成后。

### 3. 基础代码与实战验证
```text
// ============ 实验 1：静态提升 + 循环依赖的 TDZ ============
// a.mjs
import { b } from './b.mjs';
export const a = 'A';
console.log('[a] b =', b); // ReferenceError: Cannot access 'b' before initialization

// b.mjs
import { a } from './a.mjs';
export const b = 'B';
console.log('[b] a =', a);

// entry.mjs
import './a.mjs';
// 执行流：Evaluate(entry) → 依赖 a → 依赖 b → 依赖 a（Status=evaluating，命中环，直接返回）
// → 后序执行 b 的 body → 访问 a 的绑定：Link 阶段已建好间接绑定，但 const 槽位仍在 TDZ
// → 抛 ReferenceError。这证明「链接已完成」与「值已就绪」是两件独立的事。

// ============ 实验 2：live binding，不是值拷贝 ============
// counter.mjs
export let count = 0;
export function inc() { count++; }

// main.mjs
import { count, inc } from './counter.mjs';
console.log(count); // 0
inc();
console.log(count); // 1：import 绑定指向导出模块的环境槽位，读取时实时解引用
// （若写成 import * as ns，ns.count 亦然；试图 count = 1 会在解析期抛 TypeError，绑定只读）

// ============ 实验 3：import() 的 Module Map 单例语义 ============
// side.mjs
console.log('[side] evaluated'); // 只打印一次
export const v = 1;

// main2.mjs
const p1 = import('./side.js');
const p2 = import('./side.js');
console.log(p1 === p2);                       // false：两次调用生成两个独立 Promise
const [m1, m2] = await Promise.all([p1, p2]);
console.log(m1 === m2, m1.v);                 // true 1：同一 Module Record 的同一 Namespace 对象

// ============ 实验 4：求值失败被 [[EvaluationError]] 缓存 ============
// bad.mjs: throw new Error('boom');
const e1 = await import('./bad.js').catch(e => e);
const e2 = await import('./bad.js').catch(e => e);
console.log(e1 === e2); // true：Status 已为 evaluated，不再执行 body，直接返回同一错误
```

### 4. 常见误区与进阶思考
误区一：把 import() 理解为「import 的函数版」。二者处于不同阶段：静态 import 是 ModuleItem，参与 ParseModule 产出的 [[RequestedModules]]，因此在宿主加载期就被解析、链接、求值，且 specifier 被语法约束为字符串字面量；import() 是运行期表达式，只在被求值的那一刻触发 HostImportModuleDynamically，specifier 可以是任意表达式。这直接决定：import() 的目标不会出现在静态依赖图里，tree-shaking 无法跨越它分析，`import(`./locale/${n}.js`)` 这类写法只能依赖打包器的上下文模块（glob）能力来兜底，且其副作用不再受「模块图完整性」保护——错误以 Promise reject 形式漏到调用点。另一个常在代码评审中出现的错误推论是「import() 会重新执行模块」，实际上 Module Map 保证同一 Realm 内同一解析 URL 只求值一次。

误区二：认为 ESM 的循环依赖「因为 hoisting 所以没问题」，把 Link 阶段的绑定提升与 Evaluate 阶段的 TDZ 混为一谈。事实是 ESM 只解决了「绑定可提前建立」，没有解决「值可能尚未初始化」：`const/let` 在依赖方先行执行时仍处 TDZ（ReferenceError，而非 undefined），`class` 声明同理。这与 CJS 的循环表现（拿到不完整的 exports 对象、字段为 undefined）是两种不同的失败模式，也是把 CJS 代码机械迁移到 ESM 时最常见的线上事故来源。同理，静态 import 的绑定是只读的 live binding，而 CJS 的 `const x = require()` 是值快照——两者的可观察语义完全不同。

进阶思考题：同一模块内同时存在 `import './x.js'` 与 `const p = import('./x.js')`，且 x.js 含顶层 await 并在 await 之后抛错。(1) 静态 import 会使整个模块图 Evaluate 失败并触发 unhandled rejection 语义吗？(2) `p` 的 reject 与静态 import 的失败是否指向同一个错误对象，为什么？(3) 若在 x.js 的 await 之前由另一个模块再次 `import('./x.js')`，该 Promise 何时 settle，它拿到的是 [[Status]] 为 evaluating-async 的同一个 Record 还是新建的 Record？能完整答出，说明你真正掌握了 Module Map、[[AsyncEvaluation]]/[[PendingAsyncDependencyCount]] 与 [[EvaluationError]] 三者的协作方式。
