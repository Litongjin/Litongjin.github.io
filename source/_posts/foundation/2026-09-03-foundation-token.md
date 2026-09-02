---
title: "每日基础技术总结 · 2026-09-03 · Token 的本质与分词机制"
date: 2026-09-03 07:01:07
categories: [技术分享]
tags: ["技术分享", "AI 开发基础（LLM & Agent）"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-03 · Token 的本质与分词机制

## 📚 今日主题

> **Token 的本质与分词机制**（AI 开发基础（LLM & Agent））

### 1. 核心概念速览
Token 是 LLM 处理文本的最小离散单元，本质是一个从字符串到有限整数集合（词表索引）的编码结果。分词机制（Tokenizer）是确定性的字符串变换：原始文本 → 规范化 → 预切分 → 子词切分 → 词汇表ID。它解决的核心问题是如何将无限连续的符号序列映射为有限、固定维度、可被神经概率模型处理的离散输入；同时控制序列长度和词汇表规模。在整个AI体系中，Tokenizer 位于数据入口，是模型输入分布与人类文本之间的唯一形式化边界。专业工程师必须掌握它，因为上下文窗口、费用计算、推理性能、安全注入、多语言能力、模型幻觉均直接受 Token 化方式影响；不理解 Tokenizer 就无法准确解释模型行为。

### 2. 底层原理剖析
底层机制分四层：1) 规范化（Normalization）：统一大小写、Unicode NFC/NFKC、去除控制字符等，保证同一语义的文本收敛到同一 token 序列。2) 预分词（Pre-tokenization）：按空格/标点/正则规则将文本切成粗粒度块，例如 GPT 系使用 `'s|'t|\p{L}+|\p{N}|[^\s\p{L}\p{N}]+` 这类正则。3) 子词切分（Subword Segmentation）：以 BPE/WordPiece/Unigram 为代表，从训练语料统计出高频字节/字符组合，构建 merges 或概率模型；编码时贪心/动态规划地将预分词结果拆成词表中的最长/最优子词。4) 映射：将每个子词在词表中的 index 作为 token id，构成模型输入。以 BPE 为例：训练时维护词表 {0..N-1} 与 merge 规则（例如 (t, h) -> th），不断合并频次最高的相邻符号对；编码时对文本的字节序列反复应用 merge 规则；解码时按 token id 反向拼接。注意：token 不是字符、不是词、更不是语义单元；它只是频率统计与压缩的产物。与前端已有知识体系对比：JS 引擎解析 `let a = 1` 时，Scanner 产出的是 `let`、`a`、`=`、`1` 这样的词法 token，再交给 Parser 生成 AST；LLM Tokenizer 相当于 Scanner，但它不构建 AST，也不做语法分析，且它的切分规则是数据驱动而非语言规范驱动。另一个混淆点：OAuth JWT 的 token 是凭据，LLM 的 token 是符号序列单元，二者名字相同但底层完全不同，正如 Java 接口与 TS 接口虽然都叫 interface，但语义和运行时行为截然不同。

### 3. 基础代码与实战验证
```text
以下用极简 Python 实现一个可验证 BPE 核心机制的最小示例。

import re
from collections import Counter

# 1) 预分词：按空格和标点切成粗粒度 token
def pre_tokenize(text):
    return re.findall(r'\w+|\s+|[^\s\w]', text)

# 2) 训练：统计相邻符号对，反复合并频次最高的对，形成 merge 规则
def train_bpe(corpus, num_merges=10):
    words = [list(pre_tokenize(t)) for t in corpus]
    merges = []
    for _ in range(num_merges):
        pairs = Counter()
        for word in words:
            for i in range(len(word)-1):
                pairs[(word[i], word[i+1])] += 1
        if not pairs:
            break
        best = max(pairs, key=pairs.get)
        merges.append(best)
        # 将词中所有相邻 best 对合并成一个新符号
        new_words = []
        for word in words:
            new_word = []
            i = 0
            while i < len(word):
                if i < len(word)-1 and (word[i], word[i+1]) == best:
                    new_word.append(best[0] + best[1])
                    i += 2
                else:
                    new_word.append(word[i])
                    i += 1
            new_words.append(new_word)
        words = new_words
    return merges

# 3) 编码：按 merge 规则顺序合并，得到 token 序列
def encode(text, merges):
    tokens = list(pre_tokenize(text))
    for a, b in merges:
        i = 0
        while i < len(tokens)-1:
            if tokens[i] == a and tokens[i+1] == b:
                tokens[i] = a + b
                del tokens[i+1]
            else:
                i += 1
    return tokens

# 4) 验证
corpus = ['low low low low low', 'lower lower', 'newest newest']
merges = train_bpe(corpus, num_merges=10)
print(encode('lowest', merges))  # 期望输出可能是 ['low', 'est']，取决于统计频率

注释：train_bpe 中的 pairs 统计的是当前符号序列中所有相邻对的频次；max 选择最高频对作为 merge；encode 按 merge 顺序贪心合并，保证与训练时的切分一致。这个最小实现展示了 token 的本质：由语料统计驱动的、确定性的符号压缩过程，而非基于语言规则。
```

### 4. 常见误区与进阶思考
误区1：认为 token 是词或字符的等价物。实际上 token 是子词，长度可长可短，一个英文单词可能被拆成多个 token，也可能多个空格/标点被合成一个 token；不同分词器词表不同，同一文本在不同模型下的 token 数完全不可比。误区2：用字符串长度或字符数估算 token 数。中文字符通常 1 个 token，英文单词约 1.3 个 token，但特殊符号、emoji、混合文本会突变；计费与上下文窗口按 token 而非字符计算，估算错误会导致请求失败或成本超预期。思考题：为什么大模型在回答 'strawberry 里有几个 r' 时经常出错？请从 token 化机制解释：'strawberry' 在常见 BPE 词表中可能被编码成 ['st', 'raw', 'berry']，模型在注意力计算时看到的是三个 token id，而不是字符序列 s-t-r-a-w-b-e-r-r-y；它无法直接访问字符级信息，因此计数需要依赖训练时学到的隐含映射，容易出错。真正理解这一点，就能解释很多 LLM 的低级错误，也能指导你在设计 RAG 或 Agent 时避免让模型做字符级精确操作。
