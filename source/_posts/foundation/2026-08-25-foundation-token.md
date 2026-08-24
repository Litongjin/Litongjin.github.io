---
title: "每日基础技术总结 · 2026-08-25 · Token 的本质与分词机制"
date: 2026-08-25 06:56:15
categories: [技术分享]
tags: ["技术分享", "AI 开发基础（LLM & Agent）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-08-25 · Token 的本质与分词机制

## 📚 今日主题

> **Token 的本质与分词机制**（AI 开发基础（LLM & Agent））

### 1. 核心概念速览
Token 是 LLM 处理文本的最小离散单元，本质上是将连续字符序列映射为有限词汇表（vocabulary）中的整数索引。分词（Tokenization）解决的是『如何将任意字符串切分为模型可处理的符号序列』这一问题，其机制是依据预设的词典和切分规则（如 BPE、WordPiece、SentencePiece）对输入文本进行无损或有损的编码。Token 位于原始文本与模型嵌入层之间，是 LLM 输入管线的第一层抽象，决定了模型的上下文窗口上限（以 token 数计）、计算成本与语义粒度。专业工程师必须掌握它，因为 API 计费、上下文长度限制、文本预处理、多语言支持、模型微调的数据构造均直接依赖 Token 的准确理解；否则会出现截断错误、费用估算偏差、检索分块失当等工程问题。

### 2. 底层原理剖析
底层机制：现代 LLM 几乎不使用字符级或词级分词，而采用子词（subword）算法，最典型的是 Byte-Pair Encoding（BPE）。其核心逻辑是：从一个字符或字节级别的初始词表开始，统计训练语料中相邻符号对的共现频率，每次合并频率最高的符号对为一个新符号，重复直至达到目标词表大小。最终得到一个有序的合并规则集，分词时按规则贪婪地应用最长匹配或按频率优先合并。伪代码（训练阶段）：

1. 将语料预归一化（Unicode NFKC、小写化等）后，按字节或字符拆分为初始符号序列。
2. 统计所有相邻符号对的频次。
3. 选择频次最高的一对 (a, b)，将其合并为新符号 ab，加入词表。
4. 重新统计受影响的相邻符号对。
5. 重复 2-4 直到词表大小达到超参数 V。

推理/编码阶段：给定字符串，按训练学到的合并规则（或直接查词表）贪心地进行最长子串匹配，将字符串切成 token 序列，每个 token 映射到词表索引。解码是逆向查表并拼接。

与前端已有知识的对比：Token 类似于前端构建工具中的『AST 节点』或『source map 映射』——它们都是对原始文本的结构化表示。但关键差异：AST 是语法驱动、递归且保留层级关系的；Token 是统计驱动、扁平且无嵌套结构的。另一个对比：JavaScript 引擎把源码解析为 Token 流（如关键字、标识符、标点），但那是按语言文法进行确定性切分；LLM 的 Token 是数据驱动学到的切分，同一段文本在不同词表下切分结果不同。本质上，Token 更接近『压缩编码』而非『语法解析』——它试图用最少的 token 数覆盖最常见的子词，类似于在前端性能优化中用字典压缩请求体。

### 3. 基础代码与实战验证
以下是一个极简 BPE 分词器的核心实现（Python），用于验证训练与编码机制。不依赖任何深度学习框架，仅用标准库。

```python
from collections import defaultdict, Counter

# 训练阶段：学习合并规则
def train_bpe(texts, vocab_size=50):
    # 初始词表：所有字符 + 特殊符号 </w>（表示词尾）
    vocab = set()
    corpus = []
    for text in texts:
        # 在单词后添加 </w> 以标记边界，避免跨词合并
        tokens = list(text.lower()) + ['</w>']
        corpus.append(tokens)
        vocab.update(tokens)
    
    merges = []
    while len(vocab) < vocab_size:
        # 统计所有相邻符号对的频率
        pairs = Counter()
        for tokens in corpus:
            for i in range(len(tokens)-1):
                pairs[(tokens[i], tokens[i+1])] += 1
        if not pairs:
            break
        # 合并频率最高的相邻对
        best_pair = max(pairs, key=pairs.get)
        merges.append(best_pair)
        new_symbol = best_pair[0] + best_pair[1]  # 新 token 即为两符号拼接
        vocab.add(new_symbol)
        # 在语料中替换所有出现的该相邻对
        new_corpus = []
        for tokens in corpus:
            i = 0
            merged = []
            while i < len(tokens):
                if i < len(tokens)-1 and (tokens[i], tokens[i+1]) == best_pair:
                    merged.append(new_symbol)
                    i += 2
                else:
                    merged.append(tokens[i])
                    i += 1
            new_corpus.append(merged)
        corpus = new_corpus
    return merges, vocab

# 编码阶段：应用合并规则将文本转为 token 索引
def encode(text, merges, vocab):
    tokens = list(text.lower()) + ['</w>']
    # 循环应用训练时学到的合并规则（顺序即优先级）
    for a, b in merges:
        new_symbol = a + b
        i = 0
        merged = []
        while i < len(tokens):
            if i < len(tokens)-1 and tokens[i] == a and tokens[i+1] == b:
                merged.append(new_symbol)
                i += 2
            else:
                merged.append(tokens[i])
                i += 1
        tokens = merged
    # 将 token 字符串映射到词表索引（实际 LLM 中索引由词表顺序决定）
    return [list(vocab).index(t) if t in vocab else len(vocab) for t in tokens]

# 验证
texts = ['low low low', 'lowest lowest', 'newer newer']
merges, vocab = train_bpe(texts, vocab_size=20)
print('学习到的合并规则:', merges)
print('编码结果:', encode('lowest', merges, vocab))
```

注释说明：`merge` 列表保存了从字符对到新 token 的合并顺序，编码时按此顺序重新扫描序列，贪心地应用所有可能合并，最终得到 token 序列。真实 GPT 模型使用的 tokenizer（如 cl100k_base）在此基础上增加了字节级 fallback、正则预切分、特殊 token 处理等，但核心的『频率驱动子词合并』原理一致。

### 4. 常见误区与进阶思考
误区一：认为 Token 等于单词或按空格切分。这是最普遍的认知错误。实际上英语中常见单词可能被切成多个 token（如 'unbelievable' 可能切成 'un' + 'believable'），而多个常见单词组合可能被合并为一个 token（如 'ice cream' 在部分词表中是单 token）。这直接影响上下文窗口的计算——不能按字符数或单词数估算 token 数。专业工程师应使用模型的 tokenizer 进行精确计数，而非凭经验猜测。

误区二：忽略 Unicode 规范化对 token 的影响。同一文本的 Unicode 分解形式（NFC vs NFD）或大小写不同，会产生完全不同的 token 序列，进而改变模型输出的质量和语义。前端工程师常处理浏览器中的 Unicode 安全问题，但这里需要额外注意：LLM 的分词器通常对文本做 NFKC 归一化，若在预处理阶段未统一，会导致 token 浪费或语义漂移。

深度思考题：给定一个训练好的 BPE 词表，如果我们在不重新训练的情况下，把词表中所有 token 的字符串长度都翻倍（例如将 'ab' 变为 'aabb'），请问模型的推理能力会发生什么变化？为什么？这揭示了 token 与模型参数之间的什么本质关系？
