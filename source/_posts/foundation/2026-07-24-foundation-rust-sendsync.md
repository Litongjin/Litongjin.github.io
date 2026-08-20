---
title: "每日基础技术总结 · 2026-07-24 · Rust 的所有权与借用检查在并发中的 Send/Sync"
date: 2026-07-24 08:00:00
categories: [技术分享]
tags: ["技术分享", "架构与设计"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-24 · Rust 的所有权与借用检查在并发中的 Send/Sync

## 📚 今日主题

> **Rust 的所有权与借用检查在并发中的 Send/Sync**（架构与设计）

### 1. 核心概念速览
Send/Sync 是 Rust 中用于标记类型线程安全属性的两个 unsafe trait。定义：Send 表示类型的所有权可以在线程之间转移；Sync 表示类型的不可变引用（&T）可以在线程之间共享，等价于 &T: Send。其本质是所有权系统在并发维度的编译期约束——通过类型系统声明哪些值可以跨线程移动/访问，从而在编译阶段消除数据竞争，无需运行时开销。机制：编译器基于类型结构自动推导，若类型的每个字段都满足 Send/Sync，则类型自动实现对应 trait；同时允许通过 unsafe impl 手动标记，将安全检查责任交给开发者。解决的问题是：在不引入垃圾回收或锁机制的前提下，静态保证线程间数据传递和共享的安全性。在整个计算机体系中，它处于“语言类型系统 + 并发模型”的交汇点，是 Rust 实现“无畏并发”的基石。专业工程师必须掌握，因为设计并发库、自定义线程安全类型或封装 C 库时，都需要精确理解 Send/Sync 的推导规则与安全边界。

### 2. 底层原理剖析
底层原理与所有权/借用检查严格绑定。所有权转移是移动语义，当值通过 move 闭包或 thread::spawn 跨线程传递时，其所有权从一个线程移动到另一个线程，编译期保证同一时刻只有一个线程拥有该值，这是 Send 的基础。借用检查保证引用不悬垂，而 Sync 则保证共享引用不会导致数据竞争。Send/Sync 是零大小标记 trait，不包含任何方法，纯粹是类型属性的编译期“标签”。编译器推导规则可视为：
- 若 T 的所有字段都是 Send，则 T 是 Send。
- 若 T 的所有字段都是 Sync，则 T 是 Sync。
- 基础类型（i32、bool、&str 等）都是 Send + Sync。
- 裸指针 *const T / *mut T 既不是 Send 也不是 Sync，因为裸指针可以任意别名且不持有所有权。
- Rc<T> 不是 Send/Sync，因为引用计数使用非原子操作，多线程同时修改会导致数据竞争。
- RefCell<T> 是 Send（若 T: Send）但不是 Sync，因为内部可变性依赖运行时检查，不提供跨线程互斥。
- Mutex<T> 是 Sync（若 T: Send），因为锁机制提供了线程安全的内部访问。
与前端知识对比：TypeScript 的接口是结构化类型，描述“形状”，是动态类型系统的编译期约束；Send/Sync 是名号式（nominal）标记 trait，描述“线程安全能力”，与所有权/借用规则紧密结合。Java 的接口定义方法签名，线程安全依赖 synchronized 等运行时机制；Send/Sync 是编译期静态声明，无运行时开销。前端中 setTimeout 跨线程（Web Worker）传递数据需序列化，而 Rust 通过类型系统直接保证跨线程传递的内存安全，这是本质差异。

### 3. 基础代码与实战验证
以下代码验证 Send/Sync 的自动推导和编译期约束。
```rust
use std::rc::Rc;
use std::sync::Arc;

// Rc 不是 Send，因为其引用计数非原子。跨线程移动所有权时，编译器直接拒绝。
fn move_rc() {
    let rc = Rc::new(42);
    std::thread::spawn(move || {
        println!("{}", rc); // 编译错误：Rc<i32> cannot be sent between threads safely
    });
}

// Arc 是 Send + Sync，因为使用原子引用计数，可安全跨线程共享所有权。
fn move_arc() {
    let arc = Arc::new(42);
    let handle = std::thread::spawn(move || {
        println!("{}", arc); // 编译通过：Arc<i32> 满足 Send
    });
    handle.join().unwrap();
}

// 自定义结构体：包含 Rc 字段，则自动推导为 !Send 和 !Sync。
struct NotSend {
    data: Rc<i32>,
}
// 若尝试将 NotSend 发送到线程，同样编译错误。

// unsafe impl Send 可以手动标记，但需自己保证安全。
// 裸指针本身不是 Send，但包装后若保证线程独占，可标记。
struct MyPtr(*mut i32);
unsafe impl Send for MyPtr {} // 必须由开发者确保该指针不会同时被多个线程访问
```
关键注释：
- `Rc<i32>` 在跨线程传递时，编译器检查其是否实现 `Send`，因为 `Rc` 的内部计数器是 `usize` 而非原子类型，推导结果为未实现，于是产生编译错误。
- `Arc` 使用原子计数器，自动实现 `Send` 和 `Sync`，所以可安全传递。
- `unsafe impl Send` 绕过检查，但开发者必须保证不违反所有权/借用规则，否则会造成未定义行为。

### 4. 常见误区与进阶思考
常见误区一：认为 Send/Sync 是运行时检查。实际上它们是编译期静态 trait 约束，不产生任何运行时开销，所有判断在编译时完成。常见误区二：认为 Arc<T> 使 T 的所有访问都线程安全。Arc 只保证引用计数的原子性，不保证内部数据 T 的并发访问安全；若 T 是普通变量，仍需通过 Mutex 等同步原语才能修改。进阶思考题：定义一个类型 `struct Foo { ptr: *mut i32 }`，为什么需要 `unsafe impl Send` 才能在线程间传递？如果同时实现 `Sync`，如何保证对 `ptr` 的解引用不会造成数据竞争？请从所有权、借用和别名规则角度解释安全条件。
