---
title: "每日基础技术总结 · 2026-07-15 · Java 内存模型中的 volatile 与 final 的内存语义"
date: 2026-07-15 08:00:00
categories: [技术分享]
tags: ["技术分享", "编程语言底层"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-15 · Java 内存模型中的 volatile 与 final 的内存语义

## 📚 今日主题

> **Java 内存模型中的 volatile 与 final 的内存语义**（编程语言底层）

### 1. 核心概念速览
JMM 是 Java 并发编程的底层抽象模型，定义多线程读写共享变量的可见性规则。volatile 的本质：对单个 volatile 变量的写与后续读建立 happens-before 关系，并禁止编译器、JIT 与 CPU 对该变量周围指令进行非法重排；底层通过内存屏障（StoreStore/StoreLoad/LoadLoad/LoadStore）强制刷新/失效缓存行，使跨线程修改立即可见。final 的本质：为不可变对象提供初始化安全（initialization safety）；正确构造并发布后，其他线程无需同步即可看到 final 字段的最终值。底层由 JMM 禁止 final 字段的写重排到构造器返回之后，并在返回点插入 StoreStore 屏障。解决的问题是 CPU 多级缓存与主存不一致、指令流水线乱序、编译器优化导致的可见性与顺序问题。在整个计算机体系中，JMM 是并发正确性的地基，也是理解 JUC 锁、AQS、无锁队列、AI 推理引擎多线程执行器的前提。专业工程师必须掌握：否则只能靠经验猜测加不加 volatile、final 是否真的不可变，无法构建高并发下的确定性系统。

### 2. 底层原理剖析
JMM 是顺序一致性模型的一种可移植近似。具体实现：volatile 写时插入 StoreStore+StoreLoad 屏障：StoreStore 禁止它之前的普通写与它重排，保证普通写先落主存；StoreLoad 禁止它与之后的 volatile 读重排，代价最高。volatile 读时插入 LoadLoad+LoadStore 屏障：LoadLoad 禁止它与之后的普通读重排，LoadStore 禁止它与之后的普通写重排。由此，线程 A 在写 volatile 之前的所有普通写，对随后读该 volatile 的线程 B 可见，形成 happens-before 边。final 语义：JSR-133 规定构造器内对 final 字段的写不能重排到构造器返回之后，且返回前插入 StoreStore 屏障，禁止 final 写与后续的发布操作重排。对于引用类型 final 字段，该保证扩展至构造器内通过该引用可达的对象（如数组元素、内部字段），确保安全发布后整个对象图可见。需要强调：final 本身不保证引用可见，引用必须通过 volatile、synchronized、锁或 Thread.start 等安全发布机制暴露。对比前端：JavaScript 单线程事件循环，普通代码不存在跨线程共享内存可见性问题；TypeScript 的 readonly/const 只是编译期类型约束，不产生任何运行时语义。前端唯一接近 volatile 的是 SharedArrayBuffer+Atomics：Atomics.load/store 在内存屏障语义上类似 Java volatile。而 Java 接口与 TS 接口的差异提醒我们：语言关键字的作用域不同——TS 接口是纯类型系统擦除，Java 接口还参与 JVM 动态分派与常量池解析，但二者都不像 volatile/final 那样直接钳制底层内存重排；理解内存语义必须穿透语言语法，进入 JMM 与 CPU 架构层。

### 3. 基础代码与实战验证
```text
验证 volatile 可见性的最小代码：

class VolatileStop {
    private volatile boolean running = true;

    void stop() {
        running = false; // volatile 写：插入 StoreStore+StoreLoad 屏障，保证此前普通写先冲刷，且该写不能被重排到后续读之后
    }

    void loop() throws InterruptedException {
        long counter = 0;
        while (running) { // volatile 读：插入 LoadLoad+LoadStore 屏障，禁止与循环体内普通读/写重排
            counter++;
        }
        System.out.println(counter);
    }
}

去掉 volatile 时，JIT 会把 running 提升到寄存器，loop 可能永远读不到主存更新。

验证 final 初始化安全的代码：

class Point {
    private final int x;
    private final int y;
    private static volatile Point published;

    Point(int x, int y) {
        this.x = x; // 构造器内 final 写，JMM 禁止重排到构造器返回之后
        this.y = y; // 构造器返回前插入 StoreStore 屏障，防止 final 写与后续发布操作重排
    }

    static void publish() {
        published = new Point(1, 2); // 通过 volatile 引用安全发布
    }

    static void read() {
        Point p = published;
        if (p != null) {
            System.out.println(p.x + "," + p.y); // 保证读到 1,2，无需额外同步
        }
    }
}

注意：若 Point 构造器把 this 泄漏到普通静态字段或启动线程，final 初始化安全不成立，其他线程可能看到 x=0。
```

### 4. 常见误区与进阶思考
常见误区 1：把 volatile 当作原子性保证。volatile 只保证单次读写的可见性与顺序，不保证复合操作原子性。count++ 本质是 read-modify-write，多线程下会丢失更新；必须用 AtomicInteger、synchronized 或 LongAdder。原因：volatile 没有互斥能力，不能阻止多个线程同时读到旧值。
常见误区 2：认为 final 字段在构造函数中赋值后，任意发布都安全。final 的初始化安全建立在『构造期间不逃逸』前提上；如果构造器把 this 发布到静态字段、注册监听器或启动线程，其他线程可能看到 final 字段的默认值。JMM 只保证正确发布后的可见性，不纠正错误发布。
进阶思考题：设 final int[] arr 在构造器中填充元素，并通过安全发布暴露给其他线程；其他线程通过 arr[i] 读取时，JMM 是否保证读到构造器内填充的值？请从 final 引用语义与内存屏障两个层面解释，并说明这种保证与 volatile 数组的区别。
