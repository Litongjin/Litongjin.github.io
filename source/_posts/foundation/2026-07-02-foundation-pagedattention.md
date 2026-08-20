---
title: "每日基础技术总结 · 2026-07-02 · PagedAttention 的页表管理与显存碎片化"
date: 2026-07-02 08:00:00
categories: [技术分享]
tags: ["技术分享", "AI 开发基础"]
author: Litongjin
---

# 每日基础技术总结 · 2026-07-02 · PagedAttention 的页表管理与显存碎片化

## 📚 今日主题

> **PagedAttention 的页表管理与显存碎片化**（AI 开发基础）

### 1. 核心概念速览
PagedAttention 是 vLLM 等推理引擎中用于管理 KV Cache 显存的分页机制，其本质是将连续虚拟显存映射到非连续物理显存块，以页（固定大小 block）为粒度分配和释放显存。它解决的核心问题是传统显存预分配导致的内部碎片和外部碎片：推理时 KV Cache 长度动态变化，若按最大序列长度预留连续显存，会因实际长度不足造成内部碎片；若按需分配连续区段，则因各序列生命周期不同产生外部碎片且无法高效共享。机制上，它仿照操作系统虚拟内存分页，维护一张逻辑页表（block table），每个序列的 KV Cache 逻辑地址通过页表映射到物理 block，物理 block 可被多个序列共享（如并行采样、beam search 中公共前缀）。在计算机体系中，它处于推理引擎与 GPU 驱动之间的显存管理层，属于系统级优化技术。专业工程师必须掌握它，因为大模型推理的吞吐瓶颈往往不是算力而是显存容量与带宽，不理解页表管理和碎片化机制，就无法设计高并发、高吞吐的推理服务，也无法正确调优 vLLM 等框架的 block size、max num seqs 等核心参数。

### 2. 底层原理剖析
底层运行机制分为三步：
1. 显存分块（Block Pool）：在初始化时将 GPU 显存按固定 block size（如 16 tokens 的 KV 空间）划分为若干物理 block，每个 block 有唯一的物理块号。所有 block 构成空闲链表，分配时摘除，释放时回插。
2. 逻辑地址与页表映射：每个序列的 KV Cache 被抽象为一系列逻辑 block（按 token 顺序编号），逻辑 block 号通过该序列的页表（block table）映射到物理 block 号。页表是一个数组，索引为逻辑 block 号，值为物理 block 号。序列继续生成 token 时，若当前逻辑 block 已满，则从空闲链表中取一个物理 block 挂到页表尾部，完成扩容。
3. 共享与写时复制（Copy-on-Write）：当多个序列共享同一段前缀（如 beam search 中同一个父序列的多个子序列）时，它们的页表项指向相同的物理 block。当某个序列需要写入自己的新 token 到共享 block 时，触发写时复制：先分配新物理 block，拷贝原数据，再更新该序列的页表指向新 block，原共享 block 的引用计数减一。
伪代码描述：
```
struct BlockTable {
    int logical_id;
    int physical_id;
    int ref_count;  // 共享引用计数
};

class PagedAttentionAllocator {
    free_blocks: list<int>;  // 空闲物理块
    block_table: map<seq_id, vector<int>>; // 序列 → 物理块号列表
    block_size: int; // 每个block能容纳的token数

    int allocate_block() {
        assert(!free_blocks.empty());
        int b = free_blocks.pop_front();
        return b;
    }

    void free_block(int b) {
        free_blocks.push_back(b);
    }

    void append_token(seq_id, token_kv) {
        vector<int>& table = block_table[seq_id];
        int last_block_id = table.back();
        if (num_tokens_in_block(last_block_id) < block_size) {
            write_to_block(last_block_id, token_kv);
        } else {
            int new_block = allocate_block();
            write_to_block(new_block, token_kv);
            table.push_back(new_block);
        }
    }

    void fork(seq_id_parent, seq_id_child) {
        // 复制页表，共享物理块，每个共享块ref_count++
        block_table[child] = block_table[parent];
        for (int b : block_table[child]) ref_count[b]++;
    }

    void write_to_shared_block(seq_id, logical_block_idx, new_token_kv) {
        int phys = block_table[seq_id][logical_block_idx];
        if (ref_count[phys] > 1) {
            int new_phys = allocate_block();
            copy_block(phys, new_phys);
            ref_count[phys]--;
            ref_count[new_phys] = 1;
            block_table[seq_id][logical_block_idx] = new_phys;
        }
        write_to_block(new_phys, new_token_kv);
    }
};
```
与前端已有概念的异同：前端工程中的虚拟 DOM 或不可变数据结构（如 Immer 的 draft state）也采用结构共享与写时复制，但前端场景是内存对象，无物理碎片问题；而 PagedAttention 的本质是显存寻址，类似于前端中大型应用使用的 ArrayBuffer 分页池或虚拟滚动列表的 windowing 机制——都通过索引/映射来管理离散资源，但前者是系统级资源管理，后者是应用级优化。更准确的类比是 Java 的接口与 TS 的接口：它们都定义契约，但 Java 接口是编译期类型约束，TS 接口是结构类型系统，本质不同；同样，操作系统的虚拟内存页表和 PagedAttention 都使用页表概念，但前者由硬件 MMU 管理，后者由软件在 GPU 显存上实现，且 PagedAttention 的 block 大小可变、共享语义更强。

### 3. 基础代码与实战验证
由于 PagedAttention 依赖 GPU 硬件和 CUDA，无法在纯 CPU 环境运行，这里给出一个模拟显存分页和写时复制的最小 Python 实现，精确演示页表与碎片化机制：
```python
import copy

BLOCK_SIZE = 4  # 每个block容纳的token数

class BlockAllocator:
    def __init__(self, total_blocks):
        self.free = list(range(total_blocks))          # 空闲物理块号列表
        self.blocks = {}                               # 物理块号 -> 内容列表
        self.ref_count = {i: 0 for i in range(total_blocks)}

    def alloc(self):
        if not self.free:
            raise MemoryError('OOM')
        b = self.free.pop()
        self.blocks[b] = []
        self.ref_count[b] = 1
        return b

    def free(self, b):
        self.ref_count[b] -= 1
        if self.ref_count[b] == 0:
            del self.blocks[b]
            self.free.append(b)

    def fork(self, parent_blocks):
        # 共享页表：物理块引用计数增加，返回新页表
        for b in parent_blocks:
            self.ref_count[b] += 1
        return list(parent_blocks)

    def write(self, seq_blocks, logical_idx, token):
        # 写时复制：如果该逻辑块被多个序列共享，则复制
        phys = seq_blocks[logical_idx]
        if self.ref_count[phys] > 1:
            new_phys = self.alloc()
            self.blocks[new_phys] = copy.deepcopy(self.blocks[phys])
            self.ref_count[phys] -= 1
            seq_blocks[logical_idx] = new_phys
        self.blocks[seq_blocks[logical_idx]].append(token)

# 使用示例：模拟一个序列追加token和fork后写时复制
allocator = BlockAllocator(total_blocks=8)
seq1_blocks = []
# 追加5个token，会占用2个逻辑block（第1个满4，第2个1）
for i, token in enumerate(['t0','t1','t2','t3','t4']):
    if i % BLOCK_SIZE == 0:
        seq1_blocks.append(allocator.alloc())  # 新逻辑block分配物理block
    allocator.blocks[seq1_blocks[-1]].append(token)  # 写入当前物理block

print('物理块分配情况:', allocator.blocks)  # 两个物理块有数据

# fork：seq2 共享 seq1 的全部物理块
seq2_blocks = allocator.fork(seq1_blocks)
print('seq2页表:', seq2_blocks, '引用计数:', {b: allocator.ref_count[b] for b in seq2_blocks})

# seq2 写入一个新token，触发写时复制（因为共享的最后一个block ref_count=2）
allocator.write(seq2_blocks, logical_idx=1, token='new_t')
print('写时复制后 seq2页表:', seq2_blocks)
print('写时复制后 seq1页表:', seq1_blocks)
print('物理块内容:', allocator.blocks)
```
关键行解释：
- `self.free.pop()` 从空闲链表取物理块，对应 PagedAttention 的物理块分配。
- `self.ref_count[b] += 1` 在 fork 时增加引用计数，体现多序列共享同一物理显存块。
- `if self.ref_count[phys] > 1:` 检测共享，`new_phys = self.alloc()` 分配新块，`copy.deepcopy` 模拟 GPU 上的显存拷贝，然后修改页表项，这就是写时复制机制。
运行后可见：初始 seq1 占 2 个物理块；fork 后两个序列共享；写时复制后 seq2 的最后一个逻辑块指向新物理块，seq1 的物理块不变，从而在不复制整个序列的情况下隔离了数据。

### 4. 常见误区与进阶思考
误区一：认为 PagedAttention 只是简单地把显存切块，与 C 语言的 memory pool 没有区别。实际上，PagedAttention 的核心价值在于逻辑页表支持动态映射和跨序列共享，而普通内存池仅解决固定大小对象分配，无法实现不同序列间的共享前缀，也无法通过引用计数做写时复制。忽视共享机制会导致在 beam search 或 parallel sampling 场景下依然浪费显存，无法理解 vLLM 为什么比 naive continuous batching 吞吐高。
误区二：认为显存碎片化在 PagedAttention 中完全消失。实际上，它消除了内部碎片（按需分配 block）和外部碎片（物理 block 等大小无空洞），但引入了新的『页表开销』和『block 内尾部碎片』：每个序列的最后一个 block 可能未写满，如果 block size 过大，这部分内部碎片依然存在；同时，页表本身也占用显存，在极短序列场景下，页表 overhead 可能超过节省的显存。
思考题：在多序列共享同一个物理 block 时，假设该 block 的引用计数为 N，若某个序列触发写时复制，为什么新分配的 block 的引用计数初始化为 1，而不是 N？如果多个序列几乎同时触发写时复制，如何避免重复分配和拷贝？请结合页表与引用计数机制，推演最坏情况下的显存峰值，并设计一种优化策略（例如延迟写时复制或 copy-on-write 合并）来降低峰值。
