---
title: "每日基础技术总结 · 2026-07-18 · Python 对象内存布局：PyObject、类型对象与可变对象"
date: 2026-07-18 08:00:00
categories: [技术分享]
tags: ["技术分享", "编程语言底层"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-18 · Python 对象内存布局：PyObject、类型对象与可变对象

## 📚 今日主题

> **Python 对象内存布局：PyObject、类型对象与可变对象**（编程语言底层）

### 1. 核心概念速览
Python对象内存布局的核心是：所有对象都以PyObject结构为前缀，该结构包含ob_refcnt（引用计数）和ob_type（指向类型对象的指针）。类型对象本身也是对象，描述了对象的大小、操作函数和元信息。可变对象在头部之后维护一个可变的引用数组或数据区域，而不可变对象的值在创建后不可改变，但其元数据（如引用计数）仍可更新。这一机制解决了动态类型、内存回收和操作分派的统一问题。在计算机体系中，这是CPython虚拟机的对象模型基础，是所有Python程序运行时的基石。专业工程师必须掌握它，因为内存占用分析、性能优化、C扩展开发、以及理解可变/不可变语义都依赖于对这一布局的精确认识。

### 2. 底层原理剖析
底层布局：所有对象起始部分都是PyObject头，定义为 typedef struct _object { Py_ssize_t ob_refcnt; struct _typeobject *ob_type; } PyObject; 对于可变长度对象（如list、str），头部扩展为PyVarObject，增加ob_size字段。类型对象是PyTypeObject，它本身也是PyObject（其ob_type指向元类type），并包含tp_basicsize、tp_itemsize、tp_dealloc、tp_repr等函数指针。对象通过ob_type找到其类型，进而找到操作方法。可变对象（如list）的实际结构是PyListObject，包含PyObject_VAR_HEAD和指向PyObject*数组的指针ob_item，数组容量单独管理；不可变对象（如int）则根据数值大小选择缓存或内嵌数据，小整数（-5~256）被全局缓存，大整数通过ob_digit存储多精度数字。对比前端：Python的类型对象是运行时实体，类似JS的构造函数；而TS的接口是编译期约束，运行时不存在。Python中变量是名字到对象的引用，所有值都是对象，包括数字，这与JS的原始值（number、string等）不同——JS原始值直接存储，而Python对象都有引用计数头。可变性的本质区别在于对象的值区域能否原地修改：list的引用数组可改，tuple的引用数组不可改，但tuple中的元素若是list，其内容仍可变，这是深层次的引用语义。

### 3. 基础代码与实战验证
```text
import sys
import ctypes

# 定义PyObject头结构
class PyObject(ctypes.Structure):
    _fields_ = [('ob_refcnt', ctypes.c_ssize_t), ('ob_type', ctypes.c_void_p)]

# 验证整数对象：非小整数，单独分配
a = 1000
obj = PyObject.from_address(id(a))
print('refcnt:', obj.ob_refcnt)  # 引用计数为1
print('type_ptr:', hex(obj.ob_type))  # 指向int类型对象的地址

# 验证list对象：可变对象，头部多ob_size字段
lst = [1, 2]
class PyVarObject(ctypes.Structure):
    _fields_ = [('ob_refcnt', ctypes.c_ssize_t), ('ob_type', ctypes.c_void_p), ('ob_size', ctypes.c_ssize_t)]
var_obj = PyVarObject.from_address(id(lst))
print('list_size:', var_obj.ob_size)  # 元素个数为2
print('list_type:', hex(var_obj.ob_type))  # 指向list类型对象

# 修改lst，观察ob_size变化
lst.append(3)
var_obj2 = PyVarObject.from_address(id(lst))
print('new_size:', var_obj2.ob_size)  # 3，证明可变对象在同一地址上更新状态
```

### 4. 常见误区与进阶思考
误区1：认为Python变量直接存储值。实际上变量是名字，绑定到对象引用；赋值 b = a 只是让b指向同一个对象，不拷贝对象。误区2：认为不可变对象完全不变。不可变指的是对象的值（如数字大小、tuple的元素集合）不可变，但引用计数等元数据位于对象头中，会随引用操作而改变。思考题：执行 a = 1000; b = a; a += 1 后，a和b分别指向哪个对象？它们的引用计数各是多少？请从PyObject的内存布局角度解释，特别注意整数对象是否被新建，以及引用计数如何变化。
