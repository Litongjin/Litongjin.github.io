---
title: "每日基础技术总结 · 2026-09-13 · Python 小整数缓存池（Small Int Cache）的实现原理与副作用"
date: 2026-09-13 08:00:00
categories: [技术分享]
tags: ["技术分享", "编程语言底层"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-13 · Python 小整数缓存池（Small Int Cache）的实现原理与副作用

## 📚 今日主题

> **Python 小整数缓存池（Small Int Cache）的实现原理与副作用**（编程语言底层）

### 1. 核心概念速览
CPython 在启动阶段预先生成并全局缓存 [-5, 256] 区间内的整数对象（PyLongObject），此后任何对该区间整数的引用操作（字面量、算术结果、函数返回值等）都会直接复用同一个对象实例。其本质是以空间换时间：避免高频小整数在每次运算时重复分配与销毁，同时利用 Python 对象不可变性保证共享安全。该机制并非语言规范（Python 语言参考未强制要求），而是 CPython 实现层面的优化细节。它位于『Python 对象模型』和『内存管理』的交叉点——直接影响对象身份（id）、比较运算（is）、引用计数归零行为以及内存占用。专业工程师必须掌握它，因为在不经意间依赖此实现（如 is 判断小整数）会写出仅靠运气正确的代码；在嵌入式、长期运行服务、内存优化场景中，亦需明确该缓存与其余整数内存策略（如大整数 free list）的边界。

### 2. 底层原理剖析
小整数缓存的实现集中在 Objects/longobject.c 中的 static PyLongObject small_ints[NSMALLNEGINTS + NSMALLPOSINTS]，其中 NSMALLNEGINTS=5，NSMALLPOSINTS=257，即 -5、-4、...、0、...、256 共 262 个。启动时 _PyLong_Init 调用 _PyLong_New（无初始化头部的分配）创建这些对象，并通过 PyLong_FromLong 将数值写入，随后始终持有强引用，永不释放。

关键路径：任何创建整数的入口——PyLong_FromLong、PyLong_FromSsize_t、以及算术运算内部调用的 _PyLong_FromSTwo? 等——都会先检测 value 是否落在 small_ints 覆盖区间，若是则直接返回对应数组元素（实际返回 &small_ints[value - NSMALLNEGINTS] 处的 PyObject*），跳过 malloc 和无符号化的常规构造流程。

缓存机制成立的前提是 Python 整数不可变（PyLongObject 维护 ob_digit 数字段，但外部无法修改值）。因此共享同一个实例不会产生状态污染。

与前端已有概念的对比——最接近的是 V8 引擎中的 Smi（Small Integer）：V8 将 31 位以内的整数直接内联在机器字中（指针打标），不分配堆对象；但 Smi 不是缓存，而是值表示。CPython 的小整数是完整的 PyObject，占用堆空间，但被静态持有从而使任何指针引用都指向同一块内存。JavaScript 没有 is 操作符，恒等比较 === 在 Smi 上是按值（机器字）比较，因此没有暴露对象身份歧义。另一个可对比的是 Java 的 Integer 缓存（-128~127，由 IntegerCache 实现），语义上与 Python 完全一致，只是默认区间更低（可通过 -XX:AutoBoxCacheMax 调整）。Java 的 == 比较的是引用，故同样存在小整数 == 为 true、大整数 == 为 false 的陷阱——这与 Python 的 is 行为完全同构。

此处核心机制是『静态预分配 + 全局强引用 + 生成管道拦截』。代数运算产生的结果进入 PyLong_FromLong 后同样触发拦截；例如 100+100 返回小整数缓存对象，而 300+300 新建对象。注意：小整数缓存与对象 free list 是两套独立机制：free list 属于大整数（超过缓存区间）在对象销毁时回收 PyLongObject 结构体，供后续同规模大整数重用；而小整数对象本身无 free list，因为其生命周期从未被销毁。

### 3. 基础代码与实战验证
```text
import sys

# 1. 验证小整数缓存的存在：相同值，同一个对象
a = 256
b = 256
print(a is b)  # True —— CPython 中 256 在缓存区间内，PyLong_FromLong 直接返回预创建对象

c = 257
d = 257
print(c is d)  # False —— 257 超出区间，每个字面量赋值都会新建 PyLongObject（尽管数值相同）

# 2. 验证区间下界
e = -5
f = -5
print(e is f)  # True —— -5 是下界

g = -6
h = -6
print(g is h)  # False

# 3. 验证算术结果同样命中缓存
x = 100 + 156  # 100+156=256，算术内部调用 PyLong_FromLong(256)
print(x is a)   # True —— 结果复用同一个小整数对象

y = 128 + 129  # 257
print(y is c)   # False

# 4. 验证缓存对象永不释放：即使 del 所有引用，id 仍然可被下一个小整数恢复
temp = [i for i in range(-5, 257)]  # 持有所有缓存对象的引用列表
del temp  # 释放列表，但缓存对象仍被 CPython 全局持有，引用计数不会归零
print(sys.getrefcount(1)  > 100)  # True —— 1 被大量复用，引用计数高（例如循环变量、默认参数等）

# 5. 验证计算过程产生的中间值
z = (lambda: 255 + 1)()  # 结果为 256，依然命中缓存
print(z is a)  # True

# 关键点：is 比较的是对象身份（即指针），不是值相等。小整数缓存导致指针相同，非缓存区间则指针不同。
# 此代码在 CPython 3.x 下成立；PyPy、Jython 等实现可能采用不同策略，故依赖该行为不可移植。
```

### 4. 常见误区与进阶思考
误区一：认为 Python 中 is 可以用于所有小整数比较或所有整数比较。实际上 is 是身份比较，即使两个整数数值相同，只要超出缓存区间（或通过动态计算产生非缓存结果），is 就会返回 False。专业工程师应只在比较 None、布尔值或已知单例时使用 is，整数一律用 ==。即便在缓存区间，依赖 is 也是破坏抽象——语言规范未要求实现必须缓存。

误区二：误以为小整数缓存会导致内存泄漏。小整数对象数量固定（262 个），由 CPython 静态持有，不随运算增长，这是预期的内存占用，不是泄漏。真正的内存泄漏隐患在于自定义类的 __del__ 与大整数 free list 交互，或者用户代码意外持有大量大整数对象导致 free list 膨胀。同时，不要试图通过 sys.intern 或对象池思路去清空小整数缓存——CPython 没有提供接口，也不应该去改。

进阶思考题：CPython 中小整数缓存对象是静态创建的 PyLongObject，本质上是不可变对象。如果通过 ctypes 直接强制修改某个小整数对象的 ob_digit 内部数值（比如将 1 的值改为 999），那么后续代码中所有引用 1 的地方是否都会显示为 999？请进一步分析：这种操作是否会破坏缓存区间内的所有对象一致性？Python 本身防御机制（不可变）为何无法阻止？以及为何 CPython 官方仍认为该操作是未定义行为而非需要修复的 bug？
