---
title: "每日基础技术总结 · 2026-10-27 · JavaScript BigInt 与 Number 的精度边界：IEEE 754 双精度浮点数陷阱"
date: 2026-10-27 11:29:41
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-27 · JavaScript BigInt 与 Number 的精度边界：IEEE 754 双精度浮点数陷阱

## 📚 今日主题

> **JavaScript BigInt 与 Number 的精度边界：IEEE 754 双精度浮点数陷阱**（前端底层与计算机基础）

### 1. 核心概念速览
## 定义

- **Number**：ECMAScript 的 Numeric 类型之一，语义等价于 IEEE 754-2019 的 `binary64`（双精度浮点）。64 bit 布局：`s(1) | e(11) | f(52)`，指数采用 bias=1023 的移码，规格化数隐含前导 1，故**有效精度为 53 bit（1 隐式 + 52 显式）**。
- **安全整数边界**：`|n| <= 2^53 - 1`（`Number.MAX_SAFE_INTEGER = 9007199254740991`）时，区间内任意两个相邻整数映射到不同的 double，整数域与 double 域在该区间**双射**。越过该边界后 ULP（unit in the last place）从 1 跳到 2 并继续倍增，整数域出现空洞："可表示"不再等于"可区分"。
- **BigInt**（ES2020）：任意精度二进制整数，符号 + 幅值表示，无溢出、无舍入误差；`typeof 1n === 'bigint'`，是**原始类型**而非 Number 的包装对象（`new BigInt()` 抛 TypeError）。与 Number 之间**不存在任何隐式转换**，混合算术直接抛 TypeError。

## 解决什么问题

- Number 解决的是"以固定 64 bit 逼近实数（含极大/极小量级、含小数）"的问题，代价是**有限精度 + 舍入**；
- BigInt 解决的是"整数域内精确、无溢出运算"的问题，代价是**任意精度带来的堆分配与慢路径运算**，且不覆盖小数。
- 二者是互补而非替代关系：需要精确整数语义时必须退出浮点域，需要小数与量级时必须留在浮点域（并按定点/整数化策略规避误差）。

## 在整个体系中的位置

IEEE 754 是所有主流运行时的公共数值底座：JS `Number`、Java `double`、C `double`、Python `float`、数据库 `double precision`、GPU 的 FP32/FP64、以及 PyTorch/TensorFlow 的 `float32/float64/float16/bfloat16` 全部沿用同一套位布局与舍入规则。因此本知识点不是"JS 的一个坑"，而是**浮点数值语义的入口**。

## 为什么专业工程师必须掌握

1. **跨语言数据一致性**：snowflake ID、数据库 `BIGINT`、Kafka/Protobuf `int64` 一旦经 `JSON.parse` 进入 JS，末尾数字会被静默改写，链路任何一环都可能丢数据。
2. **协议边界**：JSON 只有一种数字类型（无 int64/long 区分），序列化层的类型信息必须由 IDL 或字符串承载来补齐，这是接口契约设计问题。
3. **AI/数值计算**：FP16/BF16 累加、loss scaling、梯度求和顺序敏感性、`float32` 下的 epsilon 选择，本质都是同一套 ULP 与舍入机制的直接后果。
4. **性能与语义权衡**：BigInt 走运行时堆路径无法被 JIT 内联为机器字运算，在热点路径上使用会带来数量级的开销差异。

### 2. 底层原理剖析
## 1. 位编码与求值公式

```
Number 位布局（小端内存序不代表位序，这里是逻辑位序）：
 bit 63        bit 62 .. 52        bit 51 .. 0
  s(1)          e(11, 移码)          f(52, 尾数字段)

规格化数 (1 <= e <= 2046):  v = (-1)^s * 2^(e-1023) * (1 + f / 2^52)
次正规数 (e == 0)        :  v = (-1)^s * 2^(-1022)  * (    f / 2^52)
特殊值  (e == 2047)      :  f == 0 -> ±Infinity ; f != 0 -> NaN
```

解码伪代码（无分支近似）：

```
function decode(bits64):
  s = (bits64 >> 63) & 1
  e = (bits64 >> 52) & 0x7FF
  f = bits64 & 0xF_FFFF_FFFF_FFFF
  if e == 0x7FF: return f ? NaN : (s ? -Inf : +Inf)
  if e == 0:     return (-1)^s * 2^-1022 * (f / 2^52)
  return (-1)^s * 2^(e-1023) * (1 + f / 2^52)
```

## 2. ULP 与安全整数边界的严格推导

有效精度 53 bit ⇒ 对任意规格化数，其二进制有效数字串长度为 53 位。对于值落在 `[2^e, 2^(e+1))` 的数，指数权重为 `2^e`，最低有效位权重为 `2^(e-52)`，因此：

```
ULP(v) = 2^(floor(log2|v|) - 52)

区间                ULP    相邻整数是否可区分
[2^52, 2^53)        1      是（每个整数恰好占一个 double）
[2^53, 2^54)        2      否（每隔一个整数才可表示）
[2^54, 2^55)        4      否
```

`2^53` 本身可精确表示，但它与 `2^53 + 1` 舍入到同一个 double，**舍入不可逆**。因此"安全整数"的定义不是"可精确表示"，而是"数学值与该 double 互为唯一还原"：

```
Number.isSafeInteger(n)  ⟺  n 为整数 ∧ |n| <= 2^53 - 1
                          ⟺  ToNumber(n) 后 ToNumber(n) - n === 0 且不存在第二个整数舍入到同一 double
```

## 3. 舍入模式：roundTiesToEven

IEEE 754 默认舍入为"就近取偶"，JS 不提供切换接口（`Math.fround` 也是同模式，仅降精度到 binary32）：

```
function roundTiesToEven(exactValue, ulp):
  lower = floor(exactValue / ulp) * ulp
  upper = lower + ulp
  dl = exactValue - lower ; du = upper - exactValue
  if dl < du: return lower
  if du < dl: return upper
  return (lower / ulp) 为偶数 ? lower : upper   // 平局取偶数尾数
```

`0.1` 的成因：`0.1 = 1.1001100110011..._2 × 2^-4`，尾数是**无限循环二进制小数**，第 53 位后按 roundTiesToEven 截断，落到位模式 `...01101`（末位 +1）。`0.2` 同理。二者相加后再舍入一次，误差二次累积，得到 `0.30000000000000004`。这不是 bug，是二进制浮点的必然结果——**十进制小数在二进制下通常是无限循环的**，与十进制下 1/3 无法有限表示同构。

## 4. BigInt 的运行时表示与代价

```
V8 中：
Number  -> 堆外/内联 double（SMI 为 31bit 标记整数走 tagged pointer 快路径）
BigInt  -> HeapObject，{ map, bitfield(符号/长度), digits: uintptr[] }  // 幅值按 limb 数组分段

运算路径：
 a + b (Number)  -> 单条浮点加法指令（或 SMI 整数加法），可被 JIT 内联
 A + B (BigInt)  -> 运行时 C++ 例程：对齐 limb、进位传播、分配新 HeapObject
```

由此推出工程结论：BigInt 的加法/乘法是 O(limb 数)，且每次运算分配对象，GC 压力显著；不能在紧循环里当作 Number 使用。

## 5. 类型交互规则矩阵（规范层）

```
操作                        规范动作                                          结果
1n + 1                      ToNumeric(lhs)/ToNumeric(rhs) 后类型不同           TypeError
1n == 1                     抽象相等比较：BigInt 与 Number 按数学值比较         true
1n < 2                      抽象关系比较：BigInt::lessThan(Number) 数学值比较    true
1n < 1.5                    数学值比较                                          true
JSON.stringify({a:1n})      BigInt 无序列化语义                                TypeError
Number(1n)                  允许，非 2^53 安全区间内会有精度损失                可能失真
BigInt(1.9)                 允许，仅接受整数值 Number，否则 RangeError         1n
BigInt(1.5)                 非整数                                                RangeError
buf[i] & 0xFF               Number 位运算先 ToInt32（32 bit 截断回绕）          -128..127
1n << 64n                   BigInt 无位宽上限，任意精度                       精确
Math.max(...[1n])           Math.* 只接受 Number                                TypeError
```

注意 `ToInt32` 的存在：JS 的 `& | ^ ~ << >> >>>` 对 Number 一律先按 32 bit 有符号（`>>>` 为无符号）截断，这是 C 系语言 `int` 语义的遗留，与 Number 的 53 bit 精度能力无关，是另一个独立陷阱。

## 6. 与前端工程师既有概念的对比

```
维度            JS Number        JS BigInt          Java long / C int64        Java BigDecimal / Python int
语义            浮点 binary64     任意精度整数        固定 64 bit 二进制补码        任意精度（十进制/二进制）
溢出行为        静默精度丢失       不溢出（堆耗尽）     环绕（wraparound，UB 之外）  不溢出
除法舍入        就近取偶           向零截断           向零截断                     可指定
栈/堆           内联值/ tagged     HeapObject        内联值（值类型）             堆对象
与值类型的交互  自动隐式转换       禁止隐式转换        自动隐式提升                 需显式构造
```

也就是说：**Java `long` 的"溢出"是可观测、可预期的环绕语义；JS `Number` 的"溢出"是无声的精度丢失**，后者对调试的破坏性远大于前者。TS 的 `number` 类型在类型系统层面无法表达"这是 int64"，也正是在这一点上等价于 `double`——类型标注在这里提供的是**零保证**。

### 3. 基础代码与实战验证
```text
// ============================================================
// 1. 安全整数边界与 ULP 验证
// ============================================================
console.log(Number.MAX_SAFE_INTEGER);        // 9007199254740991 = 2^53 - 1
console.log(Number.MIN_SAFE_INTEGER);        // -9007199254740991

console.log(2 ** 53);                        // 9007199254740992，指数 53，仍可精确表示
console.log(2 ** 53 + 1);                    // 9007199254740992！
// 原理：2^53+1 的有效位需要 54 bit，超出 53 bit 精度；
// roundTiesToEven 在 ulp=2 的 [2^53, 2^54) 区间内把 2^53+1 舍入到最近的偶数 2^53。
console.log(2 ** 53 + 1 === 2 ** 53);        // true，舍入不可逆
console.log(2 ** 53 + 2);                    // 9007199254740994，ULP 已变为 2
console.log(Number.isSafeInteger(2 ** 53));  // false：可表示但不可区分
console.log((2 ** 53 + 1) - (2 ** 53));      // 0，减法在已失真的输入上无法恢复信息

// ULP 递推：值落在 [2^e, 2^(e+1)) 时 ULP = 2^(e-52)
for (let e = 50; e <= 54; e++) {
  const lo = 2 ** e, hi = 2 ** (e + 1);
  console.log(e, lo, hi - lo, 2 ** (e - 52)); // 末列即该区间的 ULP
}

// ============================================================
// 2. 十进制小数的二进制不可表示性
// ============================================================
console.log(0.1 + 0.2);                      // 0.30000000000000004
console.log(0.1 + 0.2 === 0.3);              // false
console.log((0.1).toPrecision(20));          // 0.10000000000000000555（真实存储值）
console.log((0.3).toPrecision(20));          // 0.29999999999999998890
// 0.1 尾数为无限循环 1.1001100110011..._2，第 53 位后 roundTiesToEven 舍入
console.log(Number.EPSILON);                 // 2.220446049250313e-16 = 2^-52

// 浮点相等判定必须引入容差
const epsEqual = (a, b, tol = Number.EPSILON) => Math.abs(a - b) <= tol * Math.max(Math.abs(a), Math.abs(b));
console.log(epsEqual(0.1 + 0.2, 0.3));       // true

// ============================================================
// 3. 通过类型化数组视图直接读取 64 bit 位模式
// ============================================================
const ab = new ArrayBuffer(8);
const f64 = new Float64Array(ab);            // 同一块内存的 double 视图
const u64 = new BigUint64Array(ab);          // 同一块内存的无符号 64bit 整数视图

f64[0] = 0.1;
console.log(u64[0].toString(2).padStart(64, '0'));
// 0 01111111011 100110011001100110011001100110011001100110011001100110011001
//   |     |    尾数末位因舍入 +1（真实尾数为 ...011010）
//   |     指数 1019 -> 2^(1019-1023) = 2^-4
//   符号 0

f64[0] = 2 ** 53;
console.log(u64[0].toString(2));             // 10000110100 后接 52 个 0，指数 1076 -> 2^53
f64[0] = -0;
console.log(u64[0].toString(16));            // 8000000000000000，符号位独立存在 -0
console.log(Object.is(-0, 0));               // false，而 -0 === 0 为 true

// ============================================================
// 4. BigInt：独立于浮点的精确整数域
// ============================================================
console.log(2n ** 53n + 1n);                 // 9007199254740993n，精确无舍入
console.log(typeof 2n);                      // 'bigint'，原始类型
console.log(2n ** 512n);                     // 任意精度，无溢出

// 混合算术：规范在 ToNumeric 之后检测类型不一致，直接抛 TypeError
const mixed = () => { try { return 1n + 1; } catch (e) { return e.constructor.name; } };
console.log(mixed());                        // 'TypeError'，刻意阻断隐式精度丢失

// 比较运算走数学值语义
console.log(1n == 1, 1n < 2, 1n < 1.5);      // true true true

// 显式转换的边界
console.log(BigInt(2 ** 53).toString());     // 9007199254740992，Number 已精确，可转
const lossy = () => { try { return BigInt(2 ** 53 + 1); } catch (e) { return e.constructor.name; } };
console.log(lossy()); // 不抛错，但得到 9007199254740992n —— 失真在 Number 阶段已经发生，BigInt 无法回溯

// ============================================================
// 5. JSON 边界的静默数据损坏（后端 int64 / snowflake ID）
// ============================================================
const payload = '{"id":9007199254740993,"name":"x"}';
const parsed = JSON.parse(payload);
console.log(parsed.id);                      // 9007199254740992，末位被改写
console.log(parsed.id === 9007199254740993); // false
// 修复策略一：传输层用字符串承载；修复策略二：解析期以 BigInt 重建
const revived = JSON.parse(payload, (k, v) =>
  typeof v === 'number' && !Number.isSafeInteger(v) ? BigInt(v) : v
);
// 注意：此时 v 已被 Number 污染，仍需在词法层保留原始文本（如 json-bigint 的做法）才真正无损
const safe = JSON.parse('{"id":"9007199254740993"}', (k, v) =>
  k === 'id' ? BigInt(v) : v);               // 字符串承载 + 显式转 BigInt，无损
console.log(safe.id);                        // 9007199254740993n

// ============================================================
// 6. 位运算的 32 bit 截断（与 BigInt 无关的独立陷阱）
// ============================================================
console.log(0xFFFFFFFF | 0);                 // -1，ToInt32 环绕
console.log(2 ** 31 | 0);                    // -2147483648
console.log(2 ** 32 | 0);                    // 0，2^32 mod 2^32 = 0
console.log(1n << 64n);                      // 18446744073709551616n，BigInt 无位宽上限

// ============================================================
// 7. 定点化：金额等场景退出浮点域
// ============================================================
const priceCents = 1999n;                    // 以最小货币单位（分）作为整数域
console.log(priceCents * 3n);                // 5997n，精确
console.log(priceCents * 3n / 100n);         // 59n，BigInt 除法向零截断，非四舍五入
console.log(typeof (1n + 1n));               // 'bigint'，同类型运算保持 bigint
```

### 4. 常见误区与进阶思考
## 误区一：混淆"可精确表示"与"可安全还原"

常见错误认知是"只要 `Number.isInteger(x)` 为 true，就说明这个整数没丢精度"。事实上任何 `|n| > 2^53 - 1` 的整数只要进入 Number 域，失真就在**赋值/解析那一刻已经发生**，此后所有检测（`isInteger`、`toFixed`、`Math.round`）都只是在已损坏的值上做判断。典型事故链：后端 `BIGINT` 主键 → JSON 序列化（合法）→ `JSON.parse`（静默改写末位）→ 前端回传 → 更新到错误记录。整个过程无任何异常抛出。

根因在于 JSON 规范只有一种 number 类型，不含 int64/long 概念，因此**类型信息必须由传输格式或 IDL 承载，不能由 JSON 数字承载**。正确做法是在协议层把 64 bit 整数强制声明为 string，或在解析器层面（词法阶段保留原始 token）以 BigInt 重建；仅在 parse 的 reviver 里做转换是不够的，因为 reviver 拿到的 v 已经是 Number。

## 误区二：把 BigInt 当作 Number 的"更精确版本"无差别替换

BigInt 与 Number 在规范层面是三处不兼容的：

1. **禁止隐式转换**：`1n + 1` 抛 TypeError，这是刻意的设计决策——避免 BigInt 转 Number 时静默丢精度，也避免 Number 转 BigInt 时对非整数抛错。因此在 API 边界上必须显式收敛类型，类型体操（TS 无法区分 `number` 与 `bigint` 的运算重载）也会随之放大。
2. **语义差异**：BigInt 除法是**向零截断**，而 Number 的除法是精确商再舍入；`5n / 2n === 2n`，而 `5 / 2 === 2.5`。迁移金额计算逻辑时，舍入策略必须从"就近取偶"改为显式的余数处理（`(a * scale + b / 2n) / b` 之类的手工四舍五入）。
3. **生态不可用**：`Math.*`、`Date`、`JSON.stringify`、TypedArray（除 `BigInt64Array`/`BigUint64Array`）均不接收 BigInt；且 BigInt 每次算术都会分配 HeapObject，无法被 JIT 内联为机器字指令，在热路径中相对 Number 有数量级开销差，同时把精度问题转化成 GC 压力问题。

补充一点易被忽略的：`Number(2n ** 60n)` 这类显式转换**不会抛错**，会静默产出失真的 double。显式转换存在，就意味着"禁止隐式转换"并不等于"安全"。

## 思考题

`2^53` 本身可以被 binary64 精确表示，为什么 `Number.isSafeInteger(2 ** 53)` 仍然返回 `false`？请从 Number 域的"可逆性"定义出发，结合 ULP 与 roundTiesToEven，说明：

- 在 `[2^52, 2^53)` 与 `[2^53, 2^54)` 两个区间内，整数到 double 的映射分别是单射还是多对一？
- 若一个 double 的数学值 `v` 落在 `[2^53, 2^54)`，仅凭 `v` 能否唯一确定用户原本输入的整数？
- 由此推广：为什么 `Number.isSafeInteger` 的判据是 `|n| <= 2^53 - 1` 而不是 `|n| <= 2^53`，并且为什么这个界与"有效精度 53 bit"是同一件事的两种表述？
