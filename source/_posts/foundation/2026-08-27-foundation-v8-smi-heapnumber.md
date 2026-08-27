---
title: "每日基础技术总结 · 2026-08-27 · V8 中 Smi 与 HeapNumber 的指针标记"
date: 2026-08-27 08:00:00
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-27 · V8 中 Smi 与 HeapNumber 的指针标记

## 📚 今日主题

> **V8 中 Smi 与 HeapNumber 的指针标记**（前端底层与计算机基础）

### 1. 核心概念速览
V8 的指针标记（pointer tagging）是将 JS 值编码为单个机器字的底层表示技术。一个 tagged value 的低位携带类型标签：LSB=0 表示 Smi（Small Integer），LSB=1 表示 HeapObject 指针。Smi 把整数值直接编码在 tagged word 的剩余位中，不分配堆内存；HeapNumber 是 HeapObject 之一，内部存储 64 位 IEEE 754 double，用于无法由 Smi 表达的数值。该机制解决的核心问题是避免每个 number 都成为堆对象，从而减少 GC 扫描与内存分配，并使加法、比较等热路径只需一次位运算即可区分类型。它位于运行时类型表示层，与 JIT、隐藏类、GC 共同构成 JS 引擎性能地基；也是理解 WebAssembly、TensorFlow.js 等数值密集型运行时为何能避免装箱拷贝的基础。专业工程师必须掌握它，才能理解为什么 JS number 有时是 32 位整数语义、为什么大整数或小数会触发堆分配，以及优化编译器如何逃逸堆表示。

### 2. 底层原理剖析
TaggedValue 的结构本质上是把一个机器字拆成 tag 区和 payload 区。
- bit0 是 tag：0 表示 Smi，1 表示 HeapObject 指针。
- Smi 的 payload 是左移一位后的有符号整数，即 raw = int_value << 1；解码时执行 int_value = raw >> 1。
- HeapObject 的真实指针由硬件按字对齐，所以 bit0 原本为 0；VM 将 bit0 置 1 使其成为 HeapObject 标签，识别时只需 raw & 1。

对于非整数或超出 Smi 范围的整数，V8 执行 number promotion：在堆上分配一个 HeapNumber 对象，并把该对象的 tagged pointer 的 LSB 置 1 返回。HeapNumber 内部有一个 double 字段，典型布局是 [map pointer, double bits]。从语义层看，JS 的 typeof 对 Smi 与 HeapNumber 都返回 number，语言层抽象隐藏了两个 representation。

与前端已有概念的对比：
1. JS primitive 自动装箱与 HeapNumber 的差异：访问 (1.5).toFixed() 时，规范临时创建可观察的 Number 包装对象；但 HeapNumber 是 VM 内部 primitive 值的堆表示，用户拿不到引用，也不具备对象身份。
2. Java 的 int 与 Integer 的类比：Smi 类似 int 的值语义，HeapNumber 类似 Integer 的堆分配语义；但 HeapNumber 不可变、不要求可空，由 VM 自动升降级。
3. Java interface 与 TS interface 的启发：Java 的 interface 是运行时真实存在的类型，TS 的 interface 只在编译期做结构约束，运行时完全擦除；同理，JS 语言层只有一个 number 类型，而 V8 运行时内部却分裂为 Smi 与 HeapNumber 两种表示。理解这种编译期/运行时差异，才能避免被语言抽象误导。

### 3. 基础代码与实战验证
```text
以下为 C++ 风格伪代码，展示 tagged value 的编解码与加法快速路径：

using Tagged = uintptr_t;
constexpr uintptr_t kSmiTag = 0;
constexpr uintptr_t kHeapTag = 1;
constexpr uintptr_t kTagMask = 1;

inline bool IsSmi(Tagged t) {
  return (t & kTagMask) == kSmiTag;  // 一条位与指令判定值类型
}

inline int32_t SmiToInt(Tagged t) {
  return static_cast<int32_t>(t >> 1); // 算术右移还原整数值，bit0 被丢弃
}

inline Tagged IntToSmi(int32_t v) {
  return static_cast<Tagged>(static_cast<uint32_t>(v) << 1); // 左移为 tag 让出 bit0
}

// HeapNumber 构造：需要堆分配，比 Smi 昂贵
HeapNumber* MakeHeapNumber(double d) {
  HeapNumber* h = AllocateHeapNumber();
  h->set_map(HeapNumberMapRuntime());
  h->set_double_bits(bit_cast<uint64_t>(d));
  return h;
}

// 数字加法快速路径：Smi + Smi 尽量原地计算
Tagged Add(Tagged a, Tagged b) {
  if (IsSmi(a) && IsSmi(b)) {
    int32_t sum = SmiToInt(a) + SmiToInt(b);
    if (SmiRangeCheck(sum)) return IntToSmi(sum); // 结果仍为 Smi，不触发 GC
    return MakeHeapNumber(sum);                   // 溢出则 promote 为 HeapNumber
  }
  double da = TaggedToDouble(a);
  double db = TaggedToDouble(b);
  return MakeHeapNumber(da + db);                 // 小数或混合运算必然堆分配
}

// V8 调试可用（需要 --allow-natives-syntax）：
%DebugPrint(1);   // 输出类似 Smi: 1
%DebugPrint(1.5); // 输出类似 HeapNumber: 1.5
```

### 4. 常见误区与进阶思考
误区一：把 HeapNumber 等同于 JS 的 Number 对象。
HeapNumber 是 primitive number 的 VM 内部堆表示，不是 new Number() 产生的对象；new Number(1) 是一个真正的 JS Object，typeof 为 object，而 1.5 的 HeapNumber 表示在语言层仍是 primitive，typeof 为 number。两者在 GC 管理、隐藏类、方法查找上完全不同。

误区二：认为 Smi/HeapNumber 是 number 的唯一运行时形态，忽略优化编译器。
在 TurboFan 的进阶编译阶段，局部变量中的 number 可能被直接放在寄存器或栈槽中作为 raw double 或 int64，不需要 HeapObject 包装。Smi/HeapNumber 是解释器与 GC 可见的 tagged value 经典表示，不是 JIT 优化后的全部事实。

思考题：
既然 Smi 的标签在 bit0，为什么 V8 不允许 Smi 的值占满整个机器字？请推导一条 tagged value 在左移一位后，bit0 为何恒为 0；若尝试用最高位做 Smi 标签，那么 C++ 中 int32_t >> 31 对负数会得到 -1，这会对解码造成什么破坏？
