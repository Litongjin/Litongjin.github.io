---
title: "每日基础技术总结 · 2024-11-20 · 装饰器高级：带参装饰器与类装饰器"
date: 2024-11-20 20:00:00
categories: [技术分享]
tags: ["技术分享", "Python 工程化"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-11-20 · 装饰器高级：带参装饰器与类装饰器

## 📚 今日主题

> **装饰器高级：带参装饰器与类装饰器**（Python 工程化）

### 1. 核心概念速览
装饰器本质是高阶函数在 Python 中的语法糖实现，用于在不修改原函数源代码的前提下扩展其行为（AOP 切面编程）。带参装饰器解决了多实例复用时的参数化配置问题，类装饰器则利用 `__call__` 协议实现了基于类的状态保持与复杂逻辑封装。在 Python 体系中，这是元编程（Metaprogramming）的基础，也是构建可复用中间件、日志系统、权限校验等工程化工具的核心机制。对于前端工程师，它相当于 TS 中高阶组件（HOC）或 React Hook 的底层原理，但差异在于 Python 是执行时动态替换而非编译时转换。

2. 底层原理剖析：
1) 带参装饰器机制：三层嵌套结构。
   - 外层函数接收配置参数，返回真正的装饰器。
   - 中层装饰器接收被装饰对象（通常是函数），返回代理函数。
   - 内层包装函数执行实际逻辑。
   调用链：decorator(args)(func)

2) 类装饰器机制：基于描述符协议或简单的 callable 接口。
   - 装饰器本身是一个类。
   - 该类必须实现 `__init__` (接收原始对象/方法) 和 `__call__` (拦截调用)。
   - 当装饰器应用于实例方法时需注意作用域绑定问题（通常需配合 descriptors 或手动处理 self）。

3) 对比前端概念：
   - TS Decorators: JS 目前通过 Stage 3 提案支持，主要用于 Angular/Vue 等框架的类型定义和反射元数据收集，运行时行为较少依赖。
   - Python Decorators: 纯粹的运行时对象替换。TS 的 HOC 是函数组合，Python 的带参装饰器是闭包链的组合；TS 的泛型约束在编译期检查，Python 的动态类型使得装饰器可以透明地包装任何签名函数（需手动处理 *args/**kwargs）。

### 3. 基础代码与实战验证
```text
# 带参装饰器示例：超时控制
import functools
import time

def timeout(max_seconds):
    # 第一层：接收配置参数
    def decorator(func):
        # 第二层：接收被装饰函数
        @functools.wraps(func)  # 关键：保留原函数的 __name__, __doc__ 等元信息，防止调试困难
        def wrapper(*args, **kwargs):
            start = time.time()
            if time.time() - start > max_seconds:
                raise TimeoutError(f"{func.__name__} execution exceeded {max_seconds}s")
            result = func(*args, **kwargs)  # 执行原函数
            return result
        return wrapper
    return decorator

# 类装饰器示例：单例模式实现
class Singleton:
    def __init__(self, cls):
        self._cls = cls  # 保存原始类引用
        self._instance = None

    def __call__(self, *args, **kwargs):
        # 拦截实例化调用
        if self._instance is None:
            self._instance = self._cls(*args, **kwargs)
        return self._instance

@Singleton
class DatabaseConnection:
    def __init__(self): pass

conn1 = DatabaseConnection()
conn2 = DatabaseConnection()
print(conn1 is conn2)  # True
```

### 4. 常见误区与进阶思考
1) 元数据丢失：未使用 @functools.wraps 会导致 wrapped_func.__name__ 变为 'wrapper'，极大影响日志追踪、调试器和序列化库的行为。
2) 类装饰器与方法绑定的混淆：将类装饰器直接用于类方法（@method_decorator）时，由于类装饰器作用于类定义阶段，而方法装饰器作用于实例创建阶段，两者结合极易导致 `self` 绑定失败。应优先使用 `method_decorator` 适配或将逻辑移至元类（Metaclass）或继承体系。

思考题：在一个异步 Web 框架中，如何实现一个既能接收异步协程（async def），又能同步装饰普通函数（def）的通用装饰器？请从 `inspect.iscoroutinefunction` 和 `asyncio.run_coroutine_threadsafe` 的角度分析兼容策略。
