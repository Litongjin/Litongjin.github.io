---
title: "每日基础技术总结 · 2026-09-15 · TypeScript 类型推导原理"
date: 2026-09-15 07:03:10
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-15 · TypeScript 类型推导原理

## 📚 今日主题

> **TypeScript 类型推导原理**（前端底层与计算机基础）

### 1. 核心概念速览
### 1. 核心概念速览

**定义**：TypeScript 的类型推导（type inference）是 tsc 前端在 Checker 阶段对缺少显式类型标注的语法节点赋予类型的静态语义分析过程。它不是独立算法模块，而是一套**双向（bidirectional）的局部约束求解**：自底向上的 synthesis（`getTypeOfExpression`：由子节点类型合成父表达式类型）与自顶向下的 contextual typing（`getContextualType`：把期望类型下推给函数表达式参数、return 表达式、数组/对象字面量元素、泛型实参）交替执行；泛型调用另有一遍『候选收集 → 选择 → 实例化 → 用实例化签名复查实参』的两遍式推断。

**本质特征**：
- 静态且擦除：推导结果只存在于编译期，Emitter 阶段全部消失（type erasure），运行时零表示；
- 结构化（structural）：相容性由成员结构与签名决定，而非声明名；
- 局部：作用域限于表达式树与当前签名实例化，不做全程序不动点求解；不是 Hindley–Milner 的全量 unification，也没有泛化（无 let-polymorphism）；
- 与流分析耦合：Binder 构造的 flow graph 使同一引用在不同程序点拥有不同 flow type。

**在体系中的位置**：编译管线 `Scanner → Parser(AST) → Binder(符号表 + 流图) → Checker(类型/推导/收窄) → Emitter(擦除)` 中的静态语义阶段，对应编译原理的 semantic analysis + type checking。

**为什么必须掌握**：
1. API 设计权：泛型能否从实参推出，决定调用端是否被迫写显式类型参数（`pair(1, 'x')` 可推导，而 `f<T>(cb: () => T)` 在没有上下文返回类型时推不出 T）；
2. 错误定位：推导失败报在声明点还是调用点，取决于候选来自哪一侧；
3. 编译性能：类型实例化有硬上限（`instantiationDepth = 100`、`instantiationCount = 5_000_000`），深层递归条件类型会触发 `Type instantiation is excessively deep`；
4. 运行时设计约束：类型被擦除、不可反射，DI / 序列化 / 校验不能依赖类型本身，必须有独立运行时信息源（metadata、schema 对象或手写 type guard）。

### 2. 底层原理剖析
### 2. 底层原理剖析

**2.1 两类类型：declared type 与 flow type**
Binder 为每个声明建立 Symbol 时记录 declared type，同时按控制流构造 flow node。Checker 的 `getFlowTypeOfReference` 沿流图回溯，遇到 `typeof x === 'string'`、`x.kind === 'a'`、`in`、`instanceof`、`is` 谓词、赋值等节点时施加过滤，得到某程序点的 flow type；多条流在汇合点重新 join。declared type 永不被改写。

**2.2 双向推导的方向语义**
- Synthesis（自底向上）：`1 + 1 → number`；`{ k: 1 } → { k: number }`；空数组字面量在无上下文时 → `never[]`。
- Contextual typing（自顶向下）：上下文类型来源包括变量/属性标注、函数调用形参类型（含泛型实例化后的形参）、return 对应的签名返回类型、类型断言目标、数组字面量元素位置。

**2.3 泛型调用的推断算法（简化伪码）**
1. 解析调用表达式，取得签名类型参数 T1..Tn；
2. 第一遍 inference：对每个实参/形参类型对执行 inferTypes(source, target)，仅当 T 以 naked type parameter 形式出现在形参类型中才产生候选，候选按来源位置分优先级（直接形参 > 嵌套 > 返回类型位置）；
3. 候选选择：若存在所有候选的公共父类型则取之，否则取候选的 union；无候选则退到类型参数的 constraint（`extends` 上界），仍无则 `unknown`（strict 下报 implicit any）；
4. instantiate：以选定类型实参替换 T 得到具体签名；
5. 第二遍：用具体签名重新检查实参，并对函数表达式参数施加 contextual typing；只出现在返回类型位置的参数，只能在这一遍从回调返回类型获得候选；
6. 之后才做 assignability 检查。3–5 一次完成，不做跨参数的反向修正迭代。

**2.4 literal widening 与 freshness**
字面量表达式先得到 fresh literal type（`'a'`、`1`），赋给可变位置（let 绑定、可变属性）时拓宽为 `string` / `number`；`const` 绑定、`readonly` 属性、`as const` 阻止拓宽。fresh 标记另在可赋值性检查中触发 excess property check——这是附加规则，不属于结构类型系统本身。

**2.5 条件类型与 `infer` 的延迟求值**
`T extends (infer U)[] ? U : never` 中，若被检查类型 T 是未实例化的类型参数，conditional type 保持 deferred，随 T 实例化再求值；`infer` 本质是引入一个在该 conditional type true 分支可见的局部类型变量。

**2.6 与既有前端/后端概念对比**
- **TS interface vs Java interface**：Java 是 nominal typing，子类型关系由 `implements` 显式声明，运行时保留 Class 对象可反射；TS 是 structural typing，只比较成员集合与签名，类型在运行时不存在，`instanceof Interface` 不可行（必须写 type guard）。
- **TS 泛型 vs Java 泛型**：都擦除，但 Java 在使用处依赖 `Class<T>` 令牌传递运行时信息（`new T()` 不可行）；TS 无类型令牌，泛型信息零运行时表达。Java 8 的 poly expression target typing 与 contextual typing 形似，但不做表达式树的双向推断。
- **与 HM/ML 对比**：HM 用 unification 求最一般类型并支持泛化（`let id = fun x -> x` 得 `∀a. a→a`）；TS 不泛化，`const id = (x) => x` 在 strict 下直接报 implicit any，必须显式写 `<T>(x: T) => x`——这是 TS 需要大量注解的根本原因。
- **与 V8/JIT 对比**：TS 静态类型与 V8 的 hidden class（Map）+ inline cache 类型反馈完全正交，前者擦除后不进入运行时，后者由实际执行路径生成。

### 3. 基础代码与实战验证
```text
### 3. 基础代码与实战验证

编译期断言工具（纯类型，运行时零开销）：

type Equal<A, B> = (<T>() => T extends A ? 1 : 2) extends (<T>() => T extends B ? 1 : 2) ? true : false;
type Assert<T extends true> = T;

// ---- (1) widening：绑定可变性决定字面量是否被拓宽 ----
let a = 'foo';                  // 无上下文类型：字面量先取 fresh literal type，赋给可变绑定 → 拓宽为 string
type _1 = Assert<Equal<typeof a, string>>;
const b = 'foo';                // const 绑定不可变 → 不拓宽，保留字面量类型
type _2 = Assert<Equal<typeof b, 'foo'>>;
const o = { k: 1 };             // 对象字面量属性位置默认可变 → 属性类型拓宽为 number
type _3 = Assert<Equal<typeof o, { k: number }>>;

// ---- (2) contextual typing：类型自顶向下注入，参数无需注解 ----
const inc: (x: number) => number = (x) => x + 1;   // x 的 contextual type = number，来自左侧标注

// ---- (3) 泛型调用：多候选合并 ----
declare function pair<T>(a: T, b: T): [T, T];
const p = pair(1, 'x');          // 候选 {number, string}，无公共父类型 → 取 union
type _4 = Assert<Equal<typeof p, [string | number, string | number]>>;

// ---- (4) 两遍式推断：第二遍才拿得到回调返回值 ----
declare function apply<T, R>(x: T, f: (v: T) => R): R;
const r = apply({ k: 1 }, (v) => v.k + 1);
// 第一遍：T 从实参 { k: 1 } 推得 { k: number }（属性 widening），R 无候选
// 第二遍：以 T = { k: number } 实例化签名，v 经 contextual typing 得 { k: number }，回调返回 number → R = number
type _5 = Assert<Equal<typeof r, number>>;

// ---- (5) 控制流分析：declared type 与 flow type 分离 ----
function narrow(x: string | number) {
  // 此处 flow type = declared type = string | number
  if (typeof x === 'string') {
    x.toUpperCase();             // type guard 过滤流图 → 该点 flow type 收窄为 string
  } else {
    x.toFixed();                 // 另一分支 flow type 收窄为 number
  }
  // 汇合点：两条流重新 join 为 string | number
}

// ---- (6) 条件类型 + infer 的延迟求值 ----
type ElementOf<T> = T extends readonly (infer U)[] ? U : never;
type _6 = Assert<Equal<ElementOf<number[]>, number>>;   // 具体类型可立即求值
declare function first<T>(xs: T): ElementOf<T>;         // T 未实例化 → conditional type 保持 deferred，不报错
const firstNum = first([1, 2]);                         // 实例化 T = number[] → ElementOf<number[]> → number
type _7 = Assert<Equal<typeof firstNum, number>>;

// ---- (7) freshness：多余属性检查只作用于 fresh 字面量 ----
interface Point { x: number }
// @ts-expect-error 新鲜对象字面量触发 excess property check
const bad: Point = { x: 1, y: 2 };
const raw = { x: 1, y: 2 };
const ok: Point = raw;           // 非 fresh：只按结构做 width subtyping 检查，合法

// ---- (8) 擦除验证：两条语句 emit 后仅剩值层代码 ----
const typed: number = 1;
const plain = 1;

验证命令：
- `npx tsc --strict --noEmit index.ts`：任一 `Assert<Equal<...>>` 不成立会直接抛类型错误，等价于编译期单元测试；
- `npx tsc index.ts --outDir dist` 后查看产物，可见 interface、类型标注、类型参数全部被擦除。
```

### 4. 常见误区与进阶思考
### 4. 常见误区与进阶思考

**误区一：把类型推导等同于 HM 的 unification，认为它会从返回值反推参数并迭代收敛。**
实际流程是一次性的『收集候选 → 选择（公共父类型优先，否则 union）→ 实例化 → 用实例化签名复查一遍』，不存在跨参数反向修正，也不做泛化。直接后果：`const id = (x) => x` 在 `--strict` 下报 `Parameter 'x' implicitly has an 'any' type`（HM 会得 `∀a. a→a`）；`fold([], (acc, x: number) => acc.concat(x))` 会因为 `[]` 在第一遍就把 A 钉死为 `never[]` 而报 `number` 不可赋给 `never`，只能靠显式类型实参或 `as number[]` 修正，而不能指望推断回头变聪明。

**误区二：混淆三种机制的作用时机——widening（fresh literal 规范化）、contextual typing（自顶向下）、CFA（流敏感收窄）。**
典型误判：以为 `const o = { a: 1 }` 的 `o.a` 是 `1`（实际是 `number`，除非 `as const` 或 `readonly` 属性）；以为多余属性检查是结构类型系统的固有规则（它只对 fresh 字面量生效，`const raw = {x:1,y:2}` 赋给 `{x: number}` 完全合法）；以为收窄会写回 declared type（实际只是叠加 flow type，闭包内对 `let` 变量的收窄会在函数被调用时被重置）。

**进阶思考题**：

declare function fold<A, B>(init: A, f: (acc: A, x: B) => A): (xs: B[]) => A;
const g = fold([], (acc, x: number) => acc.concat(x));

问：这行代码能否通过编译？A、B 分别被推为什么？改成 `fold<number[], number>([], (acc, x) => acc.concat(x))` 之后为什么就能通过？请从『候选收集顺序 + 是否存在第二遍反向修正』的机制角度解释，而不是停留在『类型不匹配』的表层结论。
