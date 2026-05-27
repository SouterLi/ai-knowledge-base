# AI 应用开发面试主题：Embedding 与向量索引工程

## 核心概念

**Embedding** 把文本映射到向量空间，距离近通常表示语义相近。面试要补一句：语义相近 **不等于** 能回答问题——「退款条件」和「到账时间」可能向量很近，但只有后者含答案。

**相似度指标**：Cosine（文本检索最常见）、Dot Product、L2——须与 **模型训练目标** 和索引库配置一致，不能随意切换。

**精确检索 vs ANN**：数据量大后生产用近似最近邻（HNSW、IVF 等），用少量召回精度换延迟和吞吐。

**Metadata 过滤**：`tenant_id`、`acl`、`document_id`、`version` 等——企业 RAG 的安全与时效基础，不是「锦上添花」。

Embedding 与向量索引是 RAG 的 **地基**：生成再强，送错上下文仍会幻觉。

```json
{
  "chunk_id": "doc_123#008",
  "document_id": "doc_123",
  "text": "原始 chunk 文本",
  "embedding": [0.12, -0.03, 0.88],
  "metadata": {
    "title": "退款流程",
    "version": 3,
    "acl": ["support", "admin"],
    "embedding_model_version": "bge-m3-v1"
  }
}
```

---

## 核心知识点

### 1. Embedding 模型选型

看：领域/中文效果、维度与存储成本、最大输入长度、吞吐延迟、API vs 自托管、**版本稳定性**（升级通常要全量重嵌，新旧向量空间不兼容）。

### 2. Chunk 与向量的关系

chunk 太小 → 语义残缺；太大 → 向量模糊、费 token。chunk 策略要和 Embedding 最大长度、评测集一起调。

### 3. 索引类型（面试爱问权衡）

| 索引 | 特点 | 适用 |
| --- | --- | --- |
| HNSW | 召回好、查询快、**内存高** | 百万级常见生产默认 |
| IVF | 先聚类再搜簇 | 千万级以上 |
| PQ/SQ | 压缩向量、省内存 | 可接受精度损失 + rerank 补 |
| Flat | 暴力精确 | 小数据或评测基准 |

**HNSW 参数**：`M`（连接数，↑召回↑内存）、`efConstruction`（建索引质量）、`efSearch`（查询召回 vs 延迟）。

### 4. 过滤：召回前还是后？

- **权限、租户**：必须前置（或分区索引），敏感内容不能先进日志/rerank/Prompt。
- **高选择性过滤**（某知识库、文档类型）：宜前置。
- **新鲜度、点击率**：可 rerank 后置。
- 前置过滤过严 → ANN 候选不足：扩大 `efSearch`、分区索引或混合 BM25。

### 5. 混合检索（企业标配）

```text
query -> 向量 top_k=50 + BM25 top_k=50 -> 去重 -> rerank -> 取 top_n 进上下文
```

向量管语义，BM25 管错误码、SKU、API 名；**Embedding 不是搜索引擎的全部**。

### 6. 更新、删除与模型升级

- 增删改都要同步向量库；权限变更更新 metadata 或重建受影响 chunk。
- 换 Embedding 模型：**新 namespace 全量重嵌 + 离线对比 Recall@K + 灰度**，旧索引保留回滚。
- 版本字段：`document_id + chunk_index + document_version + embedding_model_version`。

### 7. 容量规划

```text
原始向量 ≈ chunk数 × 维度 × 4字节
例：1e6 × 1536 × 4 ≈ 6GB（仅向量，不含 HNSW 图、metadata、副本、双版本并存）
```

面试要说明 **索引结构会显著放大** 内存。

### 8. 召回评估

评测集含 query、金标准 chunk、难例标签。看 Recall@K、MRR、NDCG；**不要只用最终答案准确率** 判断 Embedding——答案还受 Prompt、rerank、组装影响。

对照实验：固定 Embedding 调 chunk；固定 chunk 换模型；金标准不在任何 chunk → 切分/解析问题；在 chunk 但排后面 → Embedding/索引/rerank。

```python
def search_knowledge_base(request):
    filters = {
        "tenant_id": request.tenant_id,
        "acl": {"$overlap": request.user_roles},
        "deleted": False,
    }
    # query 与文档必须用同一 embedding 模型版本
    qv = embedding_model.encode(request.query)
    v_hits = vector_store.search(qv, filters=filters, top_k=20, ef_search=128)
    k_hits = keyword_store.search(request.query, filters=filters, top_k=20)
    return reranker.rank(request.query, merge_dedupe(v_hits, k_hits))[:8]
```

### 9. 线上监控

query embedding 延迟/失败率、向量检索 P95、候选不足比例、分数分布、过滤命中率、rerank 延迟、无答案率、引用采纳率。

---

## 高频面试问题与标准答案

### 1. Embedding 和 Reranker 区别？

**标准答案：**  
Embedding 做第一阶段粗召，把海量 chunk 压成向量快速找候选，追求高 Recall 和低延迟。Reranker 对少量 query-passage 对精排，追求排序质量。生产常见「向量召 50–100，rerank 留 5–10」。

### 2. 向量很像为什么答案还错？

**标准答案：**  
可能 chunk 相似但不含答案；query 太短；领域术语 Embedding 不懂；top_k 太小；metadata 过滤误杀；切分断了上下文；rerank 或裁剪把金标准丢了。我会分层查，不一句「模型幻觉」带过。

### 3. HNSW 为什么吃内存？

**标准答案：**  
不只存向量，还存图邻居，`M` 越大边越多召回越好内存越高，再加 metadata、副本和构建临时空间，往往远大于裸向量大小。

### 4. top_k 越大越好吗？

**标准答案：**  
不是。太小漏召回，太大噪声多、rerank 和 Prompt 成本高。用 Recall@K 曲线找收益拐点，再结合 P95 延迟和成本定默认值。

### 5. Embedding 模型升级注意什么？

**标准答案：**  
新旧向量空间一般不兼容，不能混搜。新建索引全量重嵌，评测集对比 Recall/MRR/延迟，小流量灰度，保留旧索引回滚，并记录 `embedding_model_version`。

### 6. metadata 过滤为何不能只在应用层？

**标准答案：**  
先召回再过滤时，无权限内容可能已进入检索服务、日志、rerank 甚至上下文。权限要在检索层执行，保证下游组件看不到越权数据。

### 7. 向量库和 Elasticsearch 是替代关系吗？

**标准答案：**  
不完全是。向量库擅语义 ANN；ES/OpenSearch 擅 BM25、过滤、聚合和成熟运维。很多企业是向量+ES 混合，再 rerank。

### 8. 很短 query 怎么办？

**标准答案：**  
结合会话改写；BM25 补强；错误码/产品名精确匹配；引导用户补全；入口默认过滤范围。不能单靠向量相似度。

### 9. 如何判断是 Embedding 问题还是 chunk 问题？

**标准答案：**  
做对照：固定模型换切分、固定切分换模型。人工看漏召回：正确答案是否存在于某 chunk——不在是解析切分；在但排后面才是向量/索引/rerank。

### 10. HNSW 参数怎么调？

**标准答案：**  
先用 Flat 或高 `efSearch` 当基准，再调 `M`、`efConstruction`、`efSearch`，用评测集和线上 P95 一起看，不凭感觉。增大 `efSearch` 通常↑召回↑延迟。

### 11. 为什么要记 embedding_model_version？

**标准答案：**  
召回异常时要区分是文档变了、切分变了还是模型/索引变了；也支持灰度、双索引对比和回滚。

### 12. 召回层该监控什么？

**标准答案：**  
嵌入延迟、检索 P95、候选不足率、最高分分布、过滤后空结果率、rerank 耗时、无答案率、用户点击的引用片段。帮助区分「没召到」「召得慢」「召得错」「过滤太严」。

---

## 面试回答加分点

- 把向量层讲成 **召回、权限、索引性能、版本治理、评估闭环** 的系统，不是一次 `similarity_search`。
- 能说清 **HNSW 参数与 Recall/延迟/内存** 三角权衡。
- 强调 **权限前置 + 混合检索 + rerank**，体现企业实践。
- 模型升级、索引重建有 **灰度与版本字段** 意识。
- 用 **Recall@K 对照实验** 定位问题，不只调 top_k 玄学。
- 容量估算包含 **图索引与副本**，显得做过生产。
