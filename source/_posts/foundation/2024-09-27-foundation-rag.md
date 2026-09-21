---
title: "每日基础技术总结 · 2024-09-27 · RAG：检索增强生成的切片与重排"
date: 2024-09-27 20:00:00
categories: [技术分享]
tags: ["技术分享", "AI / LLM 工程实战"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-09-27 · RAG：检索增强生成的切片与重排

## 📚 今日主题

> **RAG：检索增强生成的切片与重排**（AI / LLM 工程实战）

### 1. 核心概念速览
RAG（Retrieval-Augmented Generation）中的切片（Chunking）与重排（Reranking）是解决语义检索精度瓶颈的核心工程环节。切片本质是将非结构化文档按逻辑单元离散化，以平衡信息完整性与向量相似度计算的局部性；重排则是利用交叉编码器（Cross-Encoder）等更细粒度模型对初步召回结果进行二阶排序，修正余弦相似度在稀疏或歧义场景下的度量偏差。该机制位于数据预处理管道与生成推理之间，是提升LLM事实准确性、降低幻觉率的关键基础设施。工程师必须掌握它，因为 naive 的基于分割的 RAG 会导致上下文丢失或噪声引入，直接影响最终输出的可信度。

2. 底层原理剖析：切片算法根据分隔符（Delimiter）、固定长度或语义边界切割文本，需保留元数据（如页码、章节头）以供上下文重建。关键在于 Window Overlap（滑动窗口重叠），用于维持跨片段的语义连贯性。重排阶段通常采用两阶段架构：第一阶段使用轻量级双塔模型（Bi-Encoder）进行大规模粗召回（Top-K），计算效率高但忽略查询-文档交互细节；第二阶段使用重型交叉编码器（Cross-Encoder）对 Top-K 片段进行 Query-Passage Interaction，输入为 [CLS] query passage [SEP]，输出单一相关性分数。这与前端中 TS Interface 定义契约、JS Runtime 执行逻辑类似：切片如同将大组件拆分为 Hooks 或小块状态以便管理，重排则像 Virtual DOM Diff 算法，通过精细对比（Interaction）确定最终的渲染顺序（Relevance Rank），而非仅靠初步匹配（Bi-Embedding）。

### 3. 基础代码与实战验证
```text
// Python 伪代码示例：演示基于字符切片的简单实现与重排逻辑概念

def chunk_text(text, chunk_size=500, overlap=100):
    """
    基础固定长度切片，隐含了索引重建的需求
    """
    chunks = []
    start_idx = 0
    while start_idx < len(text):
        # 提取当前块
        end_idx = min(start_idx + chunk_size, len(text))
        chunk = text[start_idx:end_idx]
        
        # 关键：保留原始位置偏移量或添加前缀/后缀以维持语义
        # 在实际工程中，这里会调用 embedding 模型存入向量数据库
        chunks.append({
            "content": chunk,
            "start_pos": start_idx,
            "end_pos": end_idx,
            "metadata": {"source": "doc.pdf", "page": calculate_page(start_idx)}
        })
        
        # 滑动窗口移动：减去重叠部分，确保上下文衔接
        start_idx += (chunk_size - overlap)
    return chunks

def rerank_candidates(query, candidates, reranker_model):
    """
    重排核心：交叉编码器的输入构造与评分
    """
    scores = []
    for item in candidates:
        # Cross-Encoder 同时注入 Query 和 Document，捕捉深层语义交互
        input_tensor = construct_cross_encoder_input(
            question=query,
            passage=item["content"]
        )
        # 输出标量概率，代表相关性强弱
        score = reranker_model.predict(input_tensor)
        scores.append((item, score))
    
    # 按得分降序排列，替换原有的向量相似度顺序
    return sorted(scores, key=lambda x: x[1], reverse=True)[:top_k]
```

### 4. 常见误区与进阶思考
1. 误区：认为切片越小越好或越大越好。过小的切片丢失上下文导致无法回答宏观问题；过大的切片引入大量无关噪声（Noise），稀释关键信息并增加推理延迟。正确做法是根据下游任务的复杂度动态调整切片策略（如 QA 型任务用段落切片，摘要型用文档切片）。\n2. 误区：忽视元数据传递。切片后若丢弃原始结构信息（如标题、层级），向量检索可能召回内容相似但语境错误的内容（例如同一术语在不同章节含义不同），重排难以完全纠正这种结构性错位。\n思考题：当你的向量数据库中存储的是 Bi-Encoder 生成的稠密向量，而重排模型是训练好的 Cross-Encoder 时，如果两者使用的 Embedding 空间不一致（即特征分布差异巨大），直接复用向量化索引进行重排采样是否可行？为什么？这揭示了预训练语言模型在迁移学习中怎样的局限性？
