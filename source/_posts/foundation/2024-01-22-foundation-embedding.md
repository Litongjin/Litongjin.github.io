---
title: "每日基础技术总结 · 2024-01-22 · Embedding 模型与向量相似度度量"
date: 2024-01-22 20:00:00
categories: [技术分享]
tags: ["技术分享", "AI / LLM 工程实战"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-01-22 · Embedding 模型与向量相似度度量

## 📚 今日主题

> **Embedding 模型与向量相似度度量**（AI / LLM 工程实战）

### 1. 核心概念速览
Embedding（嵌入）是将高维离散语义对象（如文本 token、图像像素块）映射到连续低维稠密向量空间（Vector Space）的线性或非linear变换过程。其本质是通过降维保留语义拓扑结构：语义相近的对象在向量空间中距离更近。它解决的是非结构化数据的数学化表示与检索问题，是 RAG（检索增强生成）、推荐系统和聚类算法的核心基础组件。

在 AI 体系中，Embedding 位于数据预处理层至索引层之间，充当‘语义哈希’或‘语义坐标’的角色。专业工程师必须掌握它，因为 LLM 不直接处理原始文本，而是处理 Token ID 对应的 Embedding 向量；理解 Embedding 决定了能否正确设计相似度检索、向量数据库选型及误差评估指标。

### 2. 底层原理剖析
1. 映射机制：
   - 输入：离散符号序列 (如 [W1, W2])
   - 投影：通过预训练模型（如 BERT 的最后一层隐藏状态平均池化，或 DPR 的双塔结构）将序列投影为固定长度向量 d=768/1024。
   - 归一化：通常对结果向量进行 L2 归一化，使其落在单位超球面上，以便直接使用点积计算余弦相似度。

2. 相似度度量对比：
   - 前端视角类比：TS Interface 定义静态契约（Schema），而 Embedding 定义动态几何关系（Metric）。Java 接口关注类型是否兼容，向量相似度关注语义距离是否收敛。
   - 欧氏距离 (L2 Norm)：衡量绝对空间距离，受向量模长影响大，适用于聚类。
   - 余弦相似度 (Cosine Similarity)：衡量方向夹角，忽略模长，专为高维稀疏转向稠密后的语义匹配优化，公式为 dot(A,B) / (||A|| * ||B||)。
   - 点积 (Dot Product)：若向量已归一化，点积等同于余弦相似度；效率更高，常用于大规模 ANN（近似最近邻）搜索底层实现。

3. 核心逻辑流程：
   Query -> TextEncoder -> Vector Q (Normalized)
   Index -> TextEncoder -> Vector D (Normalized) + Indexer (HNSW/IVF)
   Search: dot(Q, TopK(D)) -> Sort by Score -> Return Context

### 3. 基础代码与实战验证
```text
// 原生 Python 实现 Embedding 向量化与余弦相似度计算
// 依赖库仅用于展示原理，实际项目中会使用 Sentence-Transformers 等库
import numpy as np
from sklearn.metrics.pairwise import cosine_similarity

# 模拟两个不同语义的短句
sentences = ["量子计算机利用叠加态并行计算", "传统硬盘使用磁记录存储数据"]

# 伪代码：实际的 Embedding 过程涉及 Transformer 前向传播
# def get_embedding(text): return model.encode(text) 
# 此处用随机向量模拟已归一化的 Embedding 结果（真实场景下需下载模型）
embeddings = np.array([
    [0.8, 0.1, 0.5],  # 假设这是第一句话的 3 维简化向量
    [0.1, 0.9, 0.2]   # 假设这是第二句话的 3 维简化向量
])

# 关键步骤：L2 归一化 (使向量长度为 1，将角度差异转化为点积差异)
def normalize(vectors):
    norms = np.linalg.norm(vectors, axis=1, keepdims=True)
    return vectors / norms

normed_embeddings = normalize(embeddings)

# 计算相似度矩阵
scores = cosine_similarity(normed_embeddings)

print("归一化后向量形状:", normed_embeddings.shape) # 确保维度对齐
print("语义相似度得分:\n", scores) 

# 解析：
# scores[0][0] 是自身相似度 (应为 1.0)
# scores[0][1] 是两句话的余弦相似度
# 底层运作：(A·B)/(||A||*||B||)。若未归一化，需分别计算模长再除之。
# 性能提示：GPU 加速时，通常直接做 MatMul(matrix_a.T, matrix_b)，前提是先做一次 Batch 归一化。
```

### 4. 常见误区与进阶思考
1. 误区：认为 '数字越大越相似' 且忽略量纲。
   在未归一化的空间中，欧氏距离小不代表余弦相似度大。例如一个短文本和一个长文本（词表并集）可能欧氏距离很近，但语义方向完全不同。务必确认业务使用的是 Cosine 还是 Euclidean，并在存入向量库前统一执行 L2 归一化。

2. 误区：混淆 Token Embedding 与 Document Embedding。
   Token Embedding（如 Word2Vec）捕捉词义，缺乏上下文（Polysemy 多义词歧义）；Document Embedding（如 BERT mean-pooling 或 CLIP Image Encoder）捕捉语境和整体语义。工程上，查询与索引必须使用同一层级、同一模型的编码策略，否则会出现 'Semantically Mismatched Retrieval'。

深度思考题：
假设你正在构建一个向量搜索引擎，当 Embedding 维度从 768 提升到 3072 时，为什么暴力搜索（Brute Force）的计算复杂度增长是线性的，但存储内存消耗却是几何级数的？在高并发场景下，你是如何通过 Approximate Nearest Neighbor (ANN) 算法（如 HNSW 或 IVF-PQ）在精度（Recall）与延迟（Latency）之间做权衡的？请从图遍历与量化压缩角度解释。
