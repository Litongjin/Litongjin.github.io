---
title: "每日基础技术总结 · 2025-08-21 · Spring AOP：JDK 动态代理与 CGLIB 字节码增强"
date: 2025-08-21 20:00:00
categories: [技术分享]
tags: ["技术分享", "Java 后端与 Spring 生态"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2025-08-21 · Spring AOP：JDK 动态代理与 CGLIB 字节码增强

## 📚 今日主题

> **Spring AOP：JDK 动态代理与 CGLIB 字节码增强**（Java 后端与 Spring 生态）

### 1. 核心概念速览
Spring AOP 的核心机制依赖于动态代理，旨在实现面向切面编程（AOP），即在运行时将横切关注点（如事务、日志）与业务逻辑解耦。其本质是通过在运行时生成目标对象的代理实例，拦截方法调用并增强行为。JDK 动态代理基于接口实现，CGLIB 基于字节码增强生成子类；二者解决了静态代理代码冗余问题，实现了无侵入式的功能扩展。掌握此机制是理解 Spring 容器生命周期、Bean 作用域及事务管理底层原理的前提。

### 2. 底层原理剖析
1. JDK 动态代理：依赖 java.lang.reflect.Proxy 和 InvocationHandler。原理是利用 JVM 在内存中动态生成一个实现了指定接口的代理类，该类重写所有接口方法，并在内部委托给 InvocationHandler.invoke()。对比前端 TS/JS：类似于 TypeScript 的 Interface 仅定义契约不实现，但 JS 没有原生接口类型系统，JDK 代理类似通过 Proxy object 包装原始对象进行反射调用。\n2. CGLIB 字节码增强：依赖 net.sf.cglib.proxy.Enhancer。原理是利用 ASM 库修改字节码，为被代理类生成子类（Subclass），重写非 final 方法。对比前端：类似 React/Vue 的虚拟 DOM diff 或框架层级的继承增强，但发生在编译后/运行时类加载阶段，直接操作 JVM Class 文件结构。\n3. 选择策略：若目标对象实现接口，优先 JDK；若无接口，强制 CGLIB。Spring Boot 2.x+ 默认使用 CGLIB (spring.aop.proxy-target-class=true)。

### 3. 基础代码与实战验证
```text
// JDK 动态代理示例：验证 InvocationHandler 机制\npublic interface UserService { void add(); }
class UserServiceImpl implements UserService { public void add() { System.out.println("Original"); } }
class JdkProxyHandler implements InvocationHandler {\n    private final Object target;\n    JdkProxyHandler(Object target) { this.target = target; }\n    @Override\n    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {\n        // 核心：通过反射在运行时决定调用哪个对象的方法\n        System.out.println("Before: Log Aspect"); \n        Object result = method.invoke(target, args); // 委托执行\n        System.out.println("After: Tx Commit");\n        return result;\n    }\n}\n// 客户端调用\nUserService proxy = (UserService) Proxy.newProxyInstance(\n    UserService.class.getClassLoader(), \n    new Class[]{UserService.class}, \n    new JdkProxyHandler(new UserServiceImpl())\n);\nproxy.add(); // 实际触发的是 JdkProxyHandler.invoke
```

### 4. 常见误区与进阶思考
误区 1：自调用失效。在 Spring Bean 中，如果在同一个类内部调用被 @Transactional 标记的方法，由于直接通过 this 引用而非代理对象引用，AOP 拦截器不会生效。这是因 JDK/CGLIB 代理改变了对象引用地址，只有外部注入的代理对象才包含增强逻辑。\n误区 2：final 方法不可代理。CGLIB 基于继承重写，JDK 基于接口实现，两者均无法拦截 final 或 static 修饰的方法，因为这些方法在 JVM 层面禁止重写或被动态替换。\n思考题：当目标对象同时继承父类并实现多个接口时，Spring 如何选择代理策略？如果希望强制使用 JDK 代理而忽略 CGLIB，需要在配置或注解上做何种调整？
