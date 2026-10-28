---
title: "每日基础技术总结 · 2026-10-29 · WebAssembly 内存模型：线性内存（Linear Memory）的共享与边界检查"
date: 2026-10-29 07:06:06
categories: [技术分享]
tags: ["技术分享", "前端底层与计算机基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-10-29 · WebAssembly 内存模型：线性内存（Linear Memory）的共享与边界检查

## 📚 今日主题

> **WebAssembly 内存模型：线性内存（Linear Memory）的共享与边界检查**（前端底层与计算机基础）

### 1. 核心概念速览
WebAssembly 线性内存（linear memory）是模块声明、导入或导出的一块连续、可按字节寻址、可增长的字节数组；它是受沙箱约束的平坦地址空间，不是 Wasm 的栈、不是 GC 堆、也不是宿主对象。所有 i32.load/i32.store 等内存指令都以运行时整数地址加静态 offset 计算有效地址，并按当前内存字节长度做边界检查；越界即 trap（JS 侧表现为 WebAssembly.RuntimeError），不会读写到相邻对象。shared memory 通过 threads 提案开启 shared 标志，要求声明 maximum，通常由 SharedArrayBuffer 承载，跨线程访问需配合原子指令与 wait/notify；memory.grow 受 maximum 约束。它解决的问题是：为 C/C++/Rust 等系统语言提供可移植、确定、可沙箱化的内存目标，同时让宿主（JS）能以 ArrayBuffer/TypedArray 安全观察和共享同一片字节。它在整个体系中的位置是 Wasm 字节码/ISA 抽象与宿主运行时之间的内存抽象层，是沙箱隔离、SIMD、线程与 AI 推理算子内存布局的基础。专业工程师必须掌握它，否则无法正确理解 Wasm 的指针语义、memory.grow 导致的宿主视图失效、共享内存的同步与原子性，以及引擎如何用 guard page/信号处理优化边界检查。

### 2. 底层原理剖析
1）结构：linear memory 是一个从 0 开始编号的字节数组，默认按 64 KiB 分页（1 page = 65536 B）。模块可声明初始页数与最大页数；未声明 maximum 的普通内存可按引擎策略增长，shared memory 必须声明 maximum。地址类型默认是 i32（32 位地址空间），memory64 提案可扩展到 i64。

2）寻址与边界检查：对每条 load/store，运行时从操作数栈取得地址 addr，与指令立即数中的静态 offset 相加得到有效地址 ea。规范语义要求按数学整数计算 ea = addr + offset，不允许静默按 u32/u64 回绕；然后检查 ea + N <= current_memory_bytes，其中 N 是访问宽度（1/2/4/8/16 字节）。若超出当前长度，立即 trap；否则读写对应字节。关键点：大 offset 必须参与越界判断，例如 addr=0x2、offset=0xFFFFFFFF、N=4 必须 trap，不能回绕到低地址。

3）引擎优化：语义上每次访问都有边界检查，但实现可做边界检查消除（BCE）、把多个访问合并检查、使用 guard page + SIGSEGV 信号处理等。优化只能改变检查方式，不能改变 trap 语义；越界读不能返回 0，越界写不能静默丢弃。

4）增长：memory.grow(delta) 以页为单位增加内存，返回旧页数；非共享内存增长后，JS 侧旧 ArrayBuffer 被 detach，旧 TypedArray 视图 length 变 0，必须重新读取 memory.buffer 创建视图。Wasm 内部整数地址本身仍是线性地址空间中的偏移，数据内容按地址保留；但宿主侧旧 buffer 引用失效。共享内存由 SharedArrayBuffer 承载，增长受 maximum 约束，跨线程可见性与同步由线程规范与原子指令约束，不能按非共享 detach 语义理解。

5）与前端已有概念对比：JS 的 ArrayBuffer/TypedArray 也是字节缓冲区，但 TypedArray 越界读返回 undefined、越界写被忽略；Wasm 内存指令越界直接 trap。JS 对象有隐藏类、原型链与 GC；Wasm linear memory 是平坦字节数组，没有对象头，C 结构体靠手动偏移布局，C 指针编译后就是 i32/i64 偏移。JS 的 SharedArrayBuffer + Atomics 与 Wasm shared memory + atomic 指令对应对等，但 Wasm 原子性是指令集和内存模型的一部分。

6）伪代码：
function load32(addr, offset):
  ea = unsigned_integer(addr) + unsigned_integer(offset)   // 不回绕
  if ea + 4 > memory.byteLength: trap()
  return read_u32_le(memory.bytes, ea)

function store32(addr, offset, value):
  ea = unsigned_integer(addr) + unsigned_integer(offset)
  if ea + 4 > memory.byteLength: trap()
  write_u32_le(memory.bytes, ea, value)

### 3. 基础代码与实战验证
```text
WAT（用 wat2wasm 编译；只依赖标准 WebAssembly API）：
(module
  (memory (export "mem") 1 2)          ;; 初始 1 page = 65536 B，最大 2 pages
  (func (export "store32") (param i32 i32)
    local.get 0                        ;; 地址操作数 addr
    local.get 1                        ;; 待写入的 32 位值
    i32.store offset=0 align=4)        ;; ea = addr + 0；检查 ea+4 <= mem.size
  (func (export "load32") (param i32) (result i32)
    local.get 0
    i32.load offset=0 align=4)         ;; 越界时 trap，而不是返回 0/undefined
  (func (export "store_oob") (param i32 i32)
    local.get 0
    local.get 1
    i32.store offset=4 align=4)        ;; ea = addr + 4；addr=65532 时 ea=65536，ea+4 > 65536 -> trap
)

JS 宿主验证：
const wasmBytes = ...; // wat2wasm 上述 WAT 得到的二进制
const { instance } = await WebAssembly.instantiate(wasmBytes, {});
const mem = instance.exports.mem;
const view = new Uint8Array(mem.buffer);

instance.exports.store32(0, 0x11223344);
console.log(view[0], view[1], view[2], view[3]); // 44 33 22 11：Wasm 内存按小端存放

try {
  instance.exports.store_oob(65532, 1); // 有效地址 65536，4 字节访问越过 1 page 边界
} catch (e) {
  console.log(e instanceof WebAssembly.RuntimeError); // true：Wasm trap
}

console.log(new Uint8Array(mem.buffer)[65536]); // undefined：JS TypedArray 越界不 trap，和 Wasm 指令语义不同

const old = view;
mem.grow(1); // 非共享内存：旧 ArrayBuffer 被 detach，旧 TypedArray 视图 length 变 0
console.log(old.byteLength); // 0
const view2 = new Uint8Array(mem.buffer);
view2[65536] = 1; // 新页首字节，合法；必须重新取 mem.buffer
```

### 4. 常见误区与进阶思考
误区一：把 JS 侧 TypedArray 视图当成稳定引用。非共享 memory.grow 后旧 ArrayBuffer 会 detach，旧视图 length 变为 0；Wasm 内部线性地址和已写入数据仍按地址保留，但宿主必须重新读取 memory.buffer 建视图。另一个相关错误是以为 memory.grow 会让 Wasm 内部指针自动“搬家”；实际指针只是线性地址偏移，增长不会改变已有地址的含义（除非超过新长度才越界）。

误区二：认为 shared memory 自动保证线程安全，或认为边界检查只发生在 JS/TypedArray 层。shared 只开启 SharedArrayBuffer 与原子指令能力，并要求 maximum；跨线程读写必须用 atomic 指令和同步原语，普通 load/store 不是同步手段。边界检查是 Wasm 内存指令的语义，引擎可用 guard page/信号处理优化，但越界仍必须 trap，不能依赖越界读返回 0。

思考题：在 shared memory 场景下，线程 A 执行 memory.grow，线程 B 同时执行一条带大静态 offset 的 load；规范要求 B 的访问要么在旧边界内成功，要么 trap，不能读到新页之外的地址。引擎需要维护哪些不变量（有效地址计算不 wraparound、当前长度读取的原子性、guard region 更新顺序、增长可见性）才能同时保证边界检查正确与增长可见？
