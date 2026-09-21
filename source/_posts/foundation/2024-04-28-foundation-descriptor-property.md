---
title: "每日基础技术总结 · 2024-04-28 · 描述符（Descriptor）与 @property 底层"
date: 2024-04-28 20:00:00
categories: [技术分享]
tags: ["技术分享", "Python 工程化"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-04-28 · 描述符（Descriptor）与 @property 底层

## 📚 今日主题

> **描述符（Descriptor）与 @property 底层**（Python 工程化）

### 1. 核心概念速览
描述符（Descriptor）是实现了 descriptor protocol（__get__, __set__, __delete__）的对象，本质上是 Python 属性访问机制的钩子。它解决了‘在属性访问时插入自定义逻辑’的问题，而非依赖继承或组合。@property 是描述符协议的语法糖，其底层由 property 类实现，该类封装了 fget, fset, fdel 为对应的 __get__, __set__, __delete__ 方法。

位置与必要性：Python 的属性查找链中，数据描述符优先级高于实例 __dict__。理解此机制是掌握 Python MRO、元编程、ORM 框架底层（如 Django/SQLAlchemy 字段映射）及类型系统（Type Hints）行为的基础。前端工程师需理解其与 TypeScript Getter/Setter 的本质区别：TS Getter/Setter 是 JS 引擎层面的 V8 隐藏类或 Proxy 拦截，属运行时对象特性；Python Descriptor 是类定义阶段的元协议，属语言核心的属性解析规则。

### 2. 底层原理剖析
1. 属性解析顺序 (Attribute Lookup)：
   class_instance.attr ->
   a. Check if attr is defined in the class's MRO.
   b. If found and it has __set__ or __delete__ (Data Descriptor), call __get__ immediately.
   c. Else if instance.__dict__.has(attr), return value from __dict__.
   d. Else if non-data descriptor (__get__ only), call __get__.
   e. Raise AttributeError.

2. @property 的工作流：
   - 当你在类中写 @property，编译器创建一个 property 对象，将其存入类的 __dict__ 中对应键值（非实例 __dict__）。
   - access: obj.prop -> 触发 property.__get__ -> 调用内部存储的 fget(obj).
   - set: obj.prop = val -> 触发 property.__set__ -> 调用内部存储的 fset(obj, val). 若无 setter，抛出 AttributeError.

3. 与 TS/JS 对比：
   - JS/TS: Getter/Setter 绑定到实例原型或特定实例上，修改的是属性访问器函数本身。性能开销在于每次访问都要查原型链和执行函数。无“数据描述符”概念，所有属性平等存放在 [[Prototype]] 或实例 Own Properties 中。
   - Python: Descriptor 是类级别的策略模式。通过重写 __get__/__set__ 改变的是属性获取的行为路径。数据描述符强制覆盖实例同名属性，这是 Python 独有的强约束机制。

### 3. 基础代码与实战验证
```text
# 手动实现一个只读属性描述符，模拟 @property 行为
class ReadOnlyProperty:
    def __init__(self, func):
        self.fget = func  # 保存获取函数引用

    def __get__(self, instance, owner=None):
        if instance is None:
            return self  # 类访问时返回描述符对象本身，支持类级反射
        return self.fget(instance)  # 实例访问时，调用绑定的 getter

    def __set__(self, instance, value):
        raise AttributeError("Cannot set read-only attribute")

class User:
    def __init__(self, name):
        self._name = name  # 使用私有变量存储真实数据

    @ReadOnlyProperty
    def name(self):
        return self._name

# 验证逻辑：
# u = User("Alice")
# print(u.name) # 调用 ReadOnlyProperty.__get__ -> 返回 self._name
# try:
#     u.name = "Bob"  # 触发 ReadOnlyProperty.__set__ -> 抛出异常
# except AttributeError as e:
#     print(e)         # Output: Cannot set read-only attribute
# 关键点：user.__dict__ 中没有 'name' 键，只有 '_name'。访问 'name' 命中类级别的 ReadOnlyProperty 描述符。
```

### 4. 常见误区与进阶思考
误区 1：误以为 @property 会创建新的实例属性。实际上，property 对象存在类的 __dict__ 中，不会出现在实例 __dict__ 中。若手动赋值给已装饰的属性（未定义 setter），操作对象是描述符本身还是实例取决于是否定义了 __set__。
误区 2：混淆描述符的作用域。描述符通常在类定义阶段初始化，属于类属性。若在实例初始化时动态设置描述符行为，需确保正确挂载到类或处理 __set_name__ 钩子。

深度思考：
如果我们在子类中重新定义一个名为父类描述符的同名属性（非描述符普通变量），会发生什么？根据 Python 的属性查找链，实例属性（在实例 __dict__ 中）优先级低于数据描述符但高于非数据描述符。若父类描述符是数据描述符（有 __set__），子类同名实例属性会被屏蔽吗？请推演并解释原因。
