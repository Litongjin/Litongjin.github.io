---
title: "每日基础技术总结 · 2026-09-06 · 向量数据库：HNSW/IVF-PQ 索引原理"
date: 2026-09-06 08:00:00
categories: [技术分享]
tags: ["技术分享", "AI / LLM 工程实战"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-09-06 · 向量数据库：HNSW/IVF-PQ 索引原理

## 📚 今日主题

> **向量数据库：HNSW/IVF-PQ 索引原理**（AI / LLM 工程实战）

### 1. 核心概念速览
向量数据库是面向高维向量近似最近邻（ANN）检索的存储与索引系统，核心是将暴力线性扫描的 O(n*d) 复杂度降为对数或亚线性。HNSW（Hierarchical Navigable Small World）是基于可导航小世界图的多层图索引，通过分层跳表式结构实现从粗到细的贪心搜索，size/recall 的平衡点在图的邻接表和层数；IVF-PQ（Inverted File with Product Quantization）是倒排索引与有损压缩的复合，先聚类划分空间（粗量化），再对残差做乘积量化以压缩向量存储，本质是用码本映射替代原始向量实现内存级大规模检索。它们解决的核心问题是：在存储、延迟、召回率三者约束下，让高维 ANN 检索可扩展到十亿级。整个体系中它们位于存储引擎和查询执行引擎之间，是向量数据库的 LSM-Tree/B+Tree 等价物——决定数据如何组织、写入如何更新、查询如何快速裁剪候选集。专业工程师必须理解其原理，因为选型、调参（efConstruction/M、nlist/nprobe/m）和查询性能优化都依赖对索引内部数据结构和 IO 开销的精确认知，而非黑盒调用。

### 2. 底层原理剖析
HNSW 的底层机制：构建时每个向量作为图节点，按几何距离连接邻居。多层结构遵循幂律分布：上层节点少、边稀疏，负责长距离跳跃；底层包含全部节点，负责精确近邻。插入时从最高层开始贪心搜索，逐层下降，在每一层找到最近邻后按 M 参数决定双向连接的邻居数量，并执行启发式剪枝（选择能保持连通性的邻居）。搜索时同样从最高层入口点出发，每层执行 'beam search'（候选集+visited set），进入下一层时以上层结果作为入口，底层最终返回 topK。复杂度约 O(log N * M)，内存开销为 O(N * M * 维度无关的边存储)。图结构的核心增益是 ' navigable small world' 属性——在局部邻居之外保留少数长边，使搜索路径大约为对数跳数。

IVF-PQ 的底层机制：分两步。第一步 IVF（倒排）：用 K-Means 对全部向量聚类（nlist 个簇），每个簇对应一个倒排列表，存储属于该簇的向量 ID 和残差（原始向量减去簇中心）。查询时只扫描最近的 nprobe 个簇的倒排列表，实现候选集的粗裁剪。第二步 PQ（乘积量化）：把向量均分为 m 个子向量（段），对每一段独立做 K-Means 聚类（每个子码本 256 个码字），将每个子向量量化为对应的码字 ID。最终一个向量被编码成 m 个字节，极低内存。查询时对查询向量做同样切分，预计算查询向量到每个子码本各码字的距离（距离表），然后扫描候选向量时只需查表累加 m 个子距离，用非对称距离计算（ADC）得到近似距离。IVF-PQ 的复杂度为 O(nprobe * (nlist 均值长度) * m)。

与前端概念的对比——可类比 '懒加载' 与 '虚拟滚动' 的取舍：HNSW 类似保留完整引用关系（类似虚拟 DOM 的 fiber 树）以换取快速遍历，但牺牲写入和内存；IVF-PQ 类似图片的 WebP 有损压缩——码本相当于调色板，子段相当于压缩块，查询在压缩域内执行而无需解压全量数据。更精准的对比是：HNSW 与 '跳表（Skip List）' 的层次化查找同构，而 IVF-PQ 与 'Web Worker 中的分片再编码' 同构，用结构化压缩换存储和 IO。本质差异在于：HNSW 是纯链路索引，无信息损失（除非做量化）；IVF-PQ 是有损索引，存在 recall 损失，但支持纯内存操作。

### 3. 基础代码与实战验证
以 Python + NumPy 实现极简 IVF 索引（仅粗量化部分）验证倒排剪枝原理；PQ 部分用伪代码描述。

```python
import numpy as np

class SimpleIVF:
    def __init__(self, nlist=10, niter=10):
        self.nlist = nlist
        self.centroids = None  # 簇中心 shape [nlist, d]
        self.inverted_lists = {}  # {cluster_id: List[vector]}

    def train(self, X):
        # 用 K-Means 训练簇中心，本质是寻找 d 维空间中的 nlist 个代表点，作为倒排桶的锚点
        rng = np.random.default_rng(0)
        idx = rng.choice(len(X), self.nlist, replace=False)
        self.centroids = X[idx].copy()
        for _ in range(self.niter):
            # 分配：每个向量归属最近的簇中心（暴力计算欧氏距离）
            dists = np.linalg.norm(X[:, None, :] - self.centroids[None, :, :], axis=2)  # [N, nlist]
            assign = np.argmin(dists, axis=1)
            # 更新：重新计算每个簇的均值，使簇中心逼近数据分布质心
            for c in range(self.nlist):
                if np.sum(assign == c) > 0:
                    self.centroids[c] = X[assign == c].mean(axis=0)

    def add(self, x):
        # 插入向量：计算到所有簇中心的距离，找到最近簇，放入对应倒排列表
        d = np.linalg.norm(x - self.centroids, axis=1)
        c = int(np.argmin(d))
        self.inverted_lists.setdefault(c, []).append(x)

    def search(self, q, topk=1, nprobe=2):
        # 查询：只扫描最近的 nprobe 个簇，而非全量，这就是 ANN 的 recall/latency 折中核心
        d = np.linalg.norm(q - self.centroids, axis=1)
        nearest_clusters = np.argsort(d)[:nprobe]
        candidates = []
        for c in nearest_clusters:
            candidates.extend(self.inverted_lists.get(c, []))
        if not candidates:
            return None
        cand_arr = np.array(candidates)
        # 在候选集上精确计算距离，返回最近的一个
        dists = np.linalg.norm(q - cand_arr, axis=1)
        return cand_arr[np.argmin(dists)], dists[np.argmin(dists)]

# 验证：随机 1000 个 32 维向量，建 10 个倒排桶，查询时只扫 2 桶
X = np.random.rand(1000, 32).astype(np.float32)
ivf = SimpleIVF(nlist=10)
ivf.train(X)
for v in X:
    ivf.add(v)
q = X[0].copy()
result, dist = ivf.search(q, nprobe=2)
print('recall@1 (预知真实最近邻):', np.array_equal(result, X[0]))
```

PQ 关键伪代码：
```
# 训练部分：对向量切分为 m 段，每段独立训练 256 个码字
for s in range(m):
    sub_vectors = X[:, s*d//m : (s+1)*d//m]
    codes[s] = kmeans(sub_vectors, 256)  # 得到码本 centroids[s] shape [256, d/m]
# 编码：每个向量每段找到最近码字 ID，得到 m 个字节
vec_codes = [argmin(dist(vec_segment, centroids[s]), axis=1) for s in range(m)]
# 查询：预计算查询向量到每段码本的距离表 dist_table[s][j] = dist(q_segment, centroids[s][j])
# 候选打分：对每个候选向量，累加其 m 个码字对应的 dist_table 值，即为近似距离。
```

### 4. 常见误区与进阶思考
误区 1：将 HNSW 和 IVF-PQ 视为互斥选择，认为 'HNSW 一定优于 IVF-PQ'。实际上二者应对不同规模：亿级以下、内存充足、写入少时 HNSW 延迟更低；十亿级、内存受限时则必须 IVF-PQ 或复合索引（如 IVF-HNSW）。若盲目选择导致 OOM 或延迟严重抖动，就是没理解内存占用与查询复杂度本质。

误区 2：只调 recall 指标而忽略 '查询时的 nprobe/efSearch 对 IO 的放大效应'。nprobe 或 efSearch 增大到一定程度 recall 增益急剧递减，但扫描倒排列表或图节点的次数线性增长，导致 p99 延迟先平稳后暴增。工程上必须通过 profiling 找拐点，而非一味追高 recall。

进阶思考：HNSW 的搜索过程可看作在‘贪心下降 + beam search’上运行，而 IVF-PQ 的查询是‘两级过滤（粗聚类裁剪 + 压缩距离重排）’。如果让候选集大小相等（理论上 HNSW 访问的节点数 = IVF 扫描的向量数），哪种索引在什么维度/数据分布下 recall 更高？为什么？请从图的小世界属性（存在长边保证跳数期望低）与倒排的空间划分碎片化程度（各簇大小不均匀导致负载倾斜）出发分析。
