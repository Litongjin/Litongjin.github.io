---
title: "每日基础技术总结 · 2026-09-12 · Token 的本质与分词机制"
date: 2026-09-12 07:02:11
categories: [技术分享]
tags: ["技术分享", "AI 开发基础（LLM & Agent）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-12 · Token 的本质与分词机制

## 📚 今日主题

> **Token 的本质与分词机制**（AI 开发基础（LLM & Agent））

### 1. 核心概念速览
Token 的本质是“有限离散符号表上的原子索引”：语言模型持有一张固定容量词表 V（|V| 通常为 32k~256k），任意输入文本必须被确定性映射为该词表内符号的整数 ID 序列，模型后续全部计算（embedding 查表、attention、softmax、交叉熵损失）都发生在这个整数序列空间上。分词（tokenization）就是构建并施加这一映射的完整机制，包括预分词器、统计学习得到的合并规则（BPE/WordPiece/Unigram）以及 token 字符串到 ID 的静态查表。

它解决的问题有三层。其一，空间离散化：字符串是无穷集合，必须被压缩为有限可枚举符号空间，模型才能定义 softmax 概率分布与反向传播的梯度入口。其二，粒度权衡：字符级序列过长，attention 的 O(n²) 复杂度会放大并稀释有限上下文窗口；词级词汇量无界且跨语言不可复用；subword 级在可控词表大小下保留形态学组合能力。其三，OOV 消除：字节级 fallback（UTF-8 字节作为最小单元）保证任意 Unicode 输入都有确定编码路径，彻底消灭未知词占位符。

在体系中的位置：tokenizer 是 LLM 输入管线第一层，位于原始文本与 embedding 层之间。它不是模型权重，但其词表与合并规则在预训练期冻结，模型学到的所有条件概率都以该切分为坐标轴，因此 tokenizer 与模型权重强耦合、不可随意替换。

专业工程师必须掌握的原因：工程上，token 数直接决定计费、上下文窗口实际容量、KV cache 占用与推理时延；算法上，token 边界决定了模型感知文本的粒度，是理解算术缺陷、代码缩进错误、多语言性能差异的第一性原因。它之于 LLM 应用，相当于字符编码之于浏览器渲染管线——不掌握第一层映射，就无法准确解释任何上层异常。

### 2. 底层原理剖析
底层机制以 BPE（Byte-Pair Encoding）为主流范式，GPT 系列采用字节级 BPE。完整过程分训练与推理两阶段。

一、训练阶段（离线，纯语料统计驱动）
1. 归一化：Unicode 规范化（如 NFKC）与大小写折叠；GPT 系列将空格替换为专用字符 Ġ，使空格成为 token 内容而非分隔符，从而保留词首与缩进信息。
2. 预分词：按正则把文本切分为 word 序列；word 是后续合并不可跨越的硬边界。
3. 初始化：每个 word 展开为 UTF-8 字节序列（字节级 BPE）或字符序列，并在词尾附加结束标记 </w>。
4. 迭代合并：统计语料中所有相邻 pair 的共现频次，把频次最高的 pair 合并为一个新 token 写入词表，重写语料后重复，直到词表达到预设大小。本质是贪心的层次聚类：词表最终是一棵合并树，叶子是字节，内部节点是 subword。
5. 变体差异：WordPiece 不选最高频 pair，而选使语言模型似然增益最大的 pair（score = P(pair) / (P(a) * P(b))）；Unigram 从超大词表出发用 EM 逐轮剪除使似然损失最小的 token。三者殊途同归：最小化目标语料上的期望编码长度。

二、推理/编码阶段（确定性的贪心合并，可逆）
输入文本依次经过与训练一致的归一化、预分词、字节化，然后自左向右扫描：若当前相邻 pair 命中合并规则，则合并为长 token 并回退一位以允许链式左结合；否则前进。merges 表的插入顺序即训练优先级，先学习的规则先被构造出来，这是编码确定性与可复现的保证。解码是编码的逆：按 token 的字节展开拼接并去除边界标记，无损还原原文。

伪代码（字符级 BPE 编码核心循环）：
encode(text, merges, vocab):
    words = pre_tokenize(normalize(text))
    ids = []
    for w in words:
        seq = list(w) + ['</w>']
        i = 0
        while i < len(seq) - 1:
            if (seq[i], seq[i+1]) in merges:
                seq[i:i+2] = [merges[(seq[i], seq[i+1])]]
                i = max(0, i - 1)
            else:
                i += 1
        ids.extend(vocab.index(t) for t in seq)
    return ids

注意：以上编码循环是字符级示意；字节级 BPE 的唯一区别是初始 seq 来自 UTF-8 字节展开，且 </w> 被编码为词表内的独立字节序列。核心机制（频次驱动合并、顺序即优先级、回溯保证链式合并）完全一致。

三、与前端既有概念的对比
- 与编译器词法分析（Lexer）的异同：JS 的词法 token 由手写正则定义、类别封闭（关键字/标识符/标点），LLM 的 token 由语料频次自举、类别开放（任意 subword 字符串等价类）。共性：两者都是字符串到离散类别 ID 的映射，都是下游计算（parser/transformer）的输入边界。
- 与 V8 字符串驻留（string interning）的异同：驻留是运行时按内容哈希的等价类合并，token 词表是预训练冻结的静态索引表。共性：都把一个字符串规范化为整数引用以换取查表效率。
- 与枚举/数组下标的类比：token ID 就是词表数组下标，embedding 查表等价于 matrix[id]；但词表元素是统计聚簇而非开发者声明，高频与低频 token 的 embedding 统计质量呈长尾分布，这是模型词汇偏见的来源。
- 与你熟悉的跨语言接口对比方法同理：Java 接口是运行时的类型约束与实现契约，TS 接口是编译期的结构约束；tokenizer 与它们的本质差异是——它既非运行期约束也非编译期约束，而是“先有分布、后有类别”的统计产物，类别集合本身由语料决定，不由规范制定。

### 3. 基础代码与实战验证
```text
以下用 Python 标准库实现一个极简 BPE 全流程（训练 + 编码），不依赖任何框架，用于直接观察 token 的生成机制。

import re
from collections import Counter


def train_bpe(corpus: list[str], target_vocab: int) -> dict:
    # 预分词：按空白切分，每个词转为字符元组并附加词尾标记 '</w>'
    word_freqs: Counter[tuple[str, ...]] = Counter()
    for line in corpus:
        for w in re.findall(r'\S+', line):
            word_freqs[tuple(list(w) + ['</w>'])] += 1

    merges = {}
    # 词表大小 = 当前独立字符数 + 已合并 token 数；未达标就继续合并
    while len(set(c for w in word_freqs for c in w)) + len(merges) < target_vocab:
        pair_freq: Counter[tuple[str, str]] = Counter()
        for word, freq in word_freqs.items():
            for i in range(len(word) - 1):
                pair_freq[(word[i], word[i + 1])] += freq  # 统计全局相邻 pair 频次，这是合并的唯一驱动力
        if not pair_freq:
            break
        best = pair_freq.most_common(1)[0][0]              # 取频次最高的一对
        merged = best[0] + best[1]                          # 新 token = 两个符号的字符串拼接
        merges[best] = merged                              # 记录规则；训练顺序决定合并树层次，后续编码在此基础上贪心匹配

        new_word_freqs: Counter[tuple[str, ...]] = Counter()
        for word, freq in word_freqs.items():
            new_word = []
            i = 0
            while i < len(word):
                if i < len(word) - 1 and (word[i], word[i + 1]) == best:
                    new_word.append(merged)                # 合并后整体参与下一轮统计
                    i += 2
                else:
                    new_word.append(word[i])
                    i += 1
            new_word_freqs[tuple(new_word)] += freq
        word_freqs = new_word_freqs
    return merges


def bpe_encode(text: str, merges: dict) -> list[str]:
    seq = list(text) + ['</w>']                            # 编码起点：逐字序列 + 词尾标记
    i = 0
    while i < len(seq) - 1:
        pair = (seq[i], seq[i + 1])
        if pair in merges:                                 # 命中训练期学到的合并规则
            seq[i:i + 2] = [merges[pair]]                 # 两个符号坍缩为一个 subword token
            if i > 0:
                i -= 1                                     # 回退一位，允许新 token 与左侧继续合并
        else:
            i += 1
    return seq                                             # token 序列；经词表查表即得整数 ID


# 验证：在微型语料上训练，观察 subword 从字符流中涌现
corpus = ['low low low low low', 'lower lower lower', 'newest newest newest', 'widest widest widest']
rules = train_bpe(corpus, 30)
print(bpe_encode('lowest', rules))
# 输出是模型按已学规则的贪心切分，会随语料频次分布变化，
# 这正说明 token 是统计聚簇而非语言学词根：'low' 是否独立成 token、
# 'est' 是否合并，完全取决于训练语料中相邻字符的共现频次是否足以触发合并。
```

### 4. 常见误区与进阶思考
误区一：以“字符数/词数”估算 token 数与成本。BPE 的切分是频次统计的产物，同一文本在不同词表下切分结果不同；中文汉字在字节级 BPE 中通常一个常用汉字一个 token，英文单词则常被切成 1~4 个 subword（如 tokenization 常切为 token + ization）。任何基于字符或单词的线性外推都会在计费、上下文窗口、KV cache 估算上产生系统性偏差。正确做法是直接调用与目标模型严格配对的 tokenizer（如 GPT 系的 tiktoken / cl100k_base，或开源模型的 tokenizer.json）对真实请求逐条实测，再建立 token 分布基准。

误区二：认为 tokenizer 只是预处理壳，可以随时替换或与模型解耦。词表与合并规则在预训练时已冻结，模型的 embedding 矩阵维度与全部条件分布都以该切分为坐标轴；替换 tokenizer 等价于换掉整个输入坐标系，预训练权重不可复用。深层耦合在于：token 边界会通过注意力位置编码影响模型对数字、空格、代码缩进、多语言词根的表征质量，这正是 GPT 系模型在算术与格式任务上的系统性缺陷的底层来源。因此，针对 token 边界的 prompt 设计与输入重构（如强制空格、拆分长词）是有第一性原理依据的工程手段。

进阶思考：byte-level BPE 的编码函数在固定词表与固定合并顺序下是否是单射？即是否存在两段不同的规范化文本 t1、t2，使 encode(t1) = encode(t2)？请从解码端“token 的字节展开 + 词尾标记还原”这一逆映射出发证明或构造反例，并进一步讨论：若 tokenizer 不是单射，prompt 注入检测与敏感词绕过的失效边界在哪里；若必然是单射，其前提条件（词尾标记 </w> 的作用、字节级全覆盖的意义）分别是什么。
