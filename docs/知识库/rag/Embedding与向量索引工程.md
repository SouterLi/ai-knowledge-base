# Embedding 与向量索引工程

## 核心概念

### 1. Embedding 是什么

**Embedding（嵌入向量）** 是把文本（词、句、段落、文档）映射到固定维度稠密向量空间的技术。语义相近的文本，在向量空间中距离通常更近。RAG 检索阶段的核心动作就是：**把 query 和 chunk 都变成向量，用相似度找最相关的 chunk**。

面试必须补一句关键边界：**语义相近 ≠ 能回答问题**。例如「退款条件是什么」和「退款多久到账」向量可能很近，但只有后者 chunk 里才含「7 个工作日」这类答案。向量检索解决的是「找相关候选」，不是「保证含答案」——最终还要靠 rerank、上下文组装和生成约束。

```text
文本 "如何申请退款？"  →  Embedding 模型  →  [0.12, -0.03, 0.88, ...]  (例如 768/1024/1536 维)
文本 "退款流程说明..."  →  Embedding 模型  →  [0.11, -0.01, 0.85, ...]
                                    ↓
                          Cosine 相似度 ≈ 0.92 → 认为语义相关，进入候选
```

### 2. 相似度度量：Cosine、Dot Product、L2

检索时必须明确「用什么度量相似度」，且**与 Embedding 模型训练目标和向量库配置一致**，不能随意切换。

| 度量 | 公式直觉 | 适用场景 | 注意 |
| --- | --- | --- | --- |
| **Cosine 相似度** | 向量夹角，忽略长度 | 文本语义检索最常见 | 向量需否 L2 归一化取决于模型 |
| **内积（Dot Product）** | 方向 + 模长都影响 | 某些模型原生优化内积 | 未归一化时长文本可能占优 |
| **L2 欧氏距离** | 空间直线距离 | 部分索引默认 | 高维下区分度需结合归一化 |

**工程规则：**

- query 与 document **必须用同一 Embedding 模型、同一版本、同一预处理**（是否加 instruction prefix、是否截断）
- 换相似度或换模型后，旧索引**不能**与新 query 混搜
- 很多模型输出已 L2 归一化，此时 Cosine 与内积排序等价；未归一化时不要假设等价

### 3. 精确检索 vs 近似最近邻（ANN）

| 方式 | 原理 | 复杂度 | 生产适用 |
| --- | --- | --- | --- |
| **暴力精确（Flat/Brute Force）** | 与每条向量算距离，取 top_k | O(N) | 小数据、离线评测基准 |
| **ANN 近似检索** | HNSW、IVF、PQ 等索引结构 | 亚线性 | 百万～亿级 chunk 的生产默认 |

ANN 用**少量召回精度**换**延迟和吞吐**。面试要说清：线上 Recall 下降不一定是模型差，可能是 `efSearch` 太低、索引参数太激进、或过滤后候选池太小。

### 4. Metadata 与向量是一等公民

企业 RAG 里，向量不是孤立 float 数组，必须和 **metadata** 一起设计：

```json
{
  "chunk_id": "doc_123#008",
  "document_id": "doc_123",
  "text": "退款将在审核通过后 7 个工作日内原路返回。",
  "embedding": [0.12, -0.03, 0.88],
  "metadata": {
    "title": "退款流程",
    "section": "到账时间",
    "tenant_id": "t_001",
    "acl": ["support", "admin"],
    "document_version": 3,
    "embedding_model_version": "bge-m3-v1",
    "created_at": "2026-05-01T00:00:00Z",
    "deleted": false
  }
}
```

metadata 支撑：**租户隔离、权限过滤、按文档类型/时间筛选、增量更新、版本回滚、排障归因**。不是「锦上添花」，而是安全和运维基础。

### 5. Embedding 层在 RAG 全链路中的位置

```text
离线：文档 → 切 chunk → Embedding → 写入向量库（+ 可选 BM25 索引）
在线：query →（可选改写）→ Embedding → ANN 召回 → metadata 过滤 → 混合融合 → rerank → 上下文组装 → LLM 生成
```

**地基论：** 生成模型再强，送错上下文仍会幻觉。Embedding 与索引决定「候选池质量上限」；rerank 和 Prompt 是在此基础上精修，无法弥补「金标准 chunk 根本没进 top_k」。

---

## 核心知识点

### 1. Embedding 模型选型（完整决策框架）

选型不是「哪个榜单分高用哪个」，而要同时看 **效果、成本、延迟、维度、长度、部署方式、版本稳定性**。

**必看维度：**

| 维度 | 说明 | 面试/example |
| --- | --- | --- |
| **语义效果** | 在目标领域（中文客服、代码、医疗）的 MTEB/C-MTEB 或自建评测集 Recall@K | 通用模型 vs 领域微调模型 |
| **向量维度** | 768 / 1024 / 1536 等，直接影响存储与内存 | 1536 维 × 100 万 chunk ≈ 6GB 仅向量 |
| **最大输入长度** | 模型能 encode 的最大 token | 超过则截断，长 chunk 语义丢失 |
| **中英文/多语言** | 是否支持混合语料 | 外贸、出海场景必问 |
| **query-doc 非对称** | 是否需对 query 加 instruction prefix | bge 系列常见 `Represent this sentence...` |
| **吞吐与延迟** | API QPS、自托管 GPU batch | 在线 query embedding 通常 <50ms 目标 |
| **部署方式** | OpenAI/Cohere API vs 自托管（TEI、vLLM embedding） | 数据合规、成本、可控性 |
| **版本稳定性** | 升级是否改变向量空间 | **升级通常需全量重嵌，新旧不可混搜** |

**对称 vs 非对称 Embedding：**

- **对称**：query 和 document 用同一 encode 方式（通用句向量）
- **非对称**：query 加前缀、document 不加（或不同模板），检索模型常如此设计；**离线和在线 encode 逻辑必须一致**

```python
# 中文注释：非对称模型示例——query 与 document 编码方式不同
def embed_query(text: str, model) -> list[float]:
    return model.encode(f"Represent this query for retrieval: {text}")

def embed_document(text: str, model) -> list[float]:
    return model.encode(text)  # document 不加 query 前缀
```

**API vs 自托管：**

- **API**：零运维、按量计费、版本由厂商控制；注意数据出境与 QPS 限制
- **自托管**：数据不出域、可 batch 优化吞吐；需 GPU、模型更新与扩缩容自建

**版本升级铁律：** 换 Embedding 模型 = 换向量空间 → 新建 collection/namespace → 全量重嵌 → 评测对比 → 灰度切流 → 保留旧索引回滚。字段 `embedding_model_version` 必须入库。

---

### 2. Chunk 与向量的关系（完整理解）

**一个 chunk 对应一个向量**（常规做法）。chunk 怎么切，直接决定向量「表征什么语义」。

| chunk 策略 | 对向量的影响 | 典型问题 |
| --- | --- | --- |
| **太小**（如 128 token） | 向量只含局部短语，缺章节上下文 | 「它」指代不明，召回碎片化 |
| **太大**（如 2000 token） | 向量被多主题平均，语义模糊 | 检索精度下降，rerank/Prompt 成本高 |
| **按结构切**（标题/段落） | 向量与文档结构对齐 | PDF 表格/页眉需先清洗 |
| **父子 chunk** | 小 chunk 向量用于召回，大 chunk/父文档用于上下文 | 索引体积增加，需维护关联 |

**与 Embedding 最大长度的关系：**

- chunk 超过模型 max length → **截断** → 尾部信息进不了向量
- 应在切分阶段控制 chunk size ≤ 模型有效长度（留 margin）
- 评测时要同时调 **chunk 策略** 和 **Embedding 模型**，单变量对照

**完整调优流程：**

1. 固定 Embedding 模型，换 2～3 种 chunk size/overlap，看 Recall@10
2. 固定最优 chunk，换 1～2 个 Embedding 模型对比
3. 看 badcase：金标准答案是否存在于某 chunk（不在 → 切分/解析问题；在但排后面 → 向量/索引/rerank）

---

### 3. 向量索引类型与权衡（完整对比）

生产选型本质是 **Recall、延迟、内存、构建成本** 的三角权衡。

| 索引类型 | 原理 | 优点 | 缺点 | 适用规模 |
| --- | --- | --- | --- | --- |
| **Flat（暴力）** | 遍历算距离 | 召回最准，评测基准 | O(N)，大数据不可用 | <10 万或离线评测 |
| **HNSW** | 分层可导航小世界图 | 召回好、查询快 | **内存占用高**（图边） | 百万级常见默认 |
| **IVF** | 先 K-means 聚类，只搜相近簇 | 内存低于全图、适合大规模 | 需调 nlist/nprobe，簇内仍要算距离 | 千万级以上 |
| **PQ / SQ** | 向量量化压缩 | 省内存、可上更大规模 | 精度损失，常配合 rerank 补 | 内存极紧 + 可接受损失 |

#### HNSW 核心参数（面试高频，须讲完整）

| 参数 | 含义 | 调大效果 | 调小效果 |
| --- | --- | --- | --- |
| **M** | 每个节点最大邻居数 | 召回↑、内存↑、构建慢 | 召回↓、内存↓ |
| **efConstruction** | 建索引时的搜索宽度 | 索引质量↑、构建慢 | 图质量差、召回掉 |
| **efSearch** | 查询时的搜索宽度 | 召回↑、延迟↑ | 召回↓、延迟↓ |

**调参方法（完整步骤）：**

1. 小样本用 **Flat** 或极高 `efSearch` 作 Recall 上限基准
2. 固定业务默认 `top_k`，扫 `efSearch` 画 Recall–延迟曲线，找拐点
3. 再调 `M`、`efConstruction`，在评测集和线上 P95 一起看
4. 不要凭感觉只调 `top_k`——`top_k` 再大，ANN 图搜不到也白搭

#### IVF 补充

- **nlist**：聚类中心数，类似「分桶数量」
- **nprobe**：查询时探几个簇；nprobe↑ 召回↑ 延迟↑
- 适合数据量极大、可接受一定近似、且能定期重建索引的场景

#### PQ / 标量量化

- 把高维向量压缩为码本索引，**内存可降数倍**
- 精度有损失，企业常见做法：**PQ 粗召 + Cross-Encoder rerank 精排** 拉回质量

---

### 4. Metadata 过滤：召回前还是召回后（完整策略）

**原则：敏感过滤必须前置；业务排序可后置。**

| 过滤类型 | 示例 | 建议位置 | 原因 |
| --- | --- | --- | --- |
| **租户/权限** | `tenant_id`, `acl` | **必须检索层前置** | 无权限 chunk 不能进日志、rerank、Prompt |
| **软删除** | `deleted: false` | 前置 | 避免召回已删文档 |
| **高选择性业务** | 指定知识库、文档类型 | 宜前置 | 缩小 ANN 搜索空间 |
| **新鲜度/点击率** | 时间衰减、热度 | 可 rerank 后置 | 非安全类，作排序信号 |

**前置过滤过严的问题：** ANN 在过滤后候选池太小 → 召回不足（「空结果」或相关度低）。

**应对手段：**

- 增大 `efSearch` / `top_k`（初召）
- **分区索引**：按 `tenant_id` 或 `kb_id` 分 collection，避免全局搜再过滤
- **混合 BM25**：精确词项补向量漏召
- 监控「过滤后候选数」分布

```python
def search_knowledge_base(request):
    # 中文注释：权限必须在向量检索层执行，不能先召回再在应用层 discard
    filters = {
        "tenant_id": request.tenant_id,
        "acl": {"$overlap": request.user_roles},
        "deleted": False,
    }
    # query 与 document 必须使用同一 embedding 模型版本
    qv = embedding_model.encode(
        embed_query(request.query),  # 非对称模型注意 query 前缀
        model_version=request.embedding_model_version,
    )
    v_hits = vector_store.search(
        qv, filters=filters, top_k=50, ef_search=128
    )
    k_hits = keyword_store.search(request.query, filters=filters, top_k=50)
    merged = merge_dedupe(v_hits, k_hits)  # 按 chunk_id 去重
    return reranker.rank(request.query, merged)[:8]
```

**为何不能只在应用层过滤：** 先召回再过滤时，无权限内容可能已进入检索服务日志、rerank 模型、LLM 上下文，造成**合规与泄露风险**。

---

### 5. 混合检索（完整链路）

单向量检索的盲区：

- **语义换说法**：向量擅长
- **错误码、SKU、API 名、人名、产品型号**：BM25/关键词更稳
- **极短 query**：向量信号弱

**企业标配流水线：**

```text
用户 query
  →（可选）Query 改写 / 多轮补全
  → 向量检索 top_k=50（ANN + metadata 过滤）
  → BM25 / 关键词检索 top_k=50（同过滤条件）
  → 按 chunk_id 去重合并
  →（可选）分数归一化 + 加权融合：score = α * vector_score + (1-α) * bm25_score
  → Cross-Encoder / LLM Reranker 精排
  → 取 top_n（如 5～10）进 Prompt 上下文
```

```python
def hybrid_merge(vector_hits, keyword_hits, alpha=0.5):
    """中文注释：RRF 或加权融合；chunk_id 去重"""
    by_id = {}
    for h in vector_hits:
        by_id[h.chunk_id] = {"chunk_id": h.chunk_id, "v_score": h.score, "k_score": 0.0}
    for h in keyword_hits:
        if h.chunk_id in by_id:
            by_id[h.chunk_id]["k_score"] = h.score
        else:
            by_id[h.chunk_id] = {"chunk_id": h.chunk_id, "v_score": 0.0, "k_score": h.score}
    for item in by_id.values():
        item["final"] = alpha * item["v_score"] + (1 - alpha) * item["k_score"]
    return sorted(by_id.values(), key=lambda x: x["final"], reverse=True)
```

**Embedding 不是搜索引擎的全部**——面试要能画出「向量 + 关键词 + rerank」三层结构。

---

### 6. Embedding 与 Reranker 的分工（完整对比）

| 阶段 | 模型类型 | 输入规模 | 目标 | 典型延迟 |
| --- | --- | --- | --- | --- |
| **Embedding 召回** | Bi-Encoder，query/doc 分别编码 | 百万～亿 chunk | 高 Recall、低延迟 | 毫秒～十毫秒级（ANN） |
| **Rerank 精排** | Cross-Encoder，query+doc 拼接 | 几十～几百候选 | 高 Precision、排序准 | 候选数 × 单次推理 |

**生产默认：** 向量召 **50～100**，rerank 留 **5～10** 进上下文。  
若跳过 rerank：初召噪声更容易进 Prompt，幻觉与「答非所问」上升。

---

### 7. 增量更新、删除与模型升级（完整 SOP）

#### 日常增删改

| 操作 | 向量库动作 | 注意 |
| --- | --- | --- |
| **新增文档** | 切 chunk → embed → insert | 同时更新 BM25 索引 |
| **修改文档** | 按 `document_id` **删旧 chunk** → 写新 chunk | 不能只 update 文本不更新向量 |
| **删除文档** | 删向量 + 删关键词索引 + metadata 标记 | 软删需过滤 `deleted: false` |
| **权限变更** | update metadata 或重建受影响 chunk | acl 变更要及时生效 |

#### Embedding 模型升级（完整流程）

```text
1. 新建 namespace/collection（如 kb_v2_bge-m3）
2. 离线全量重嵌所有 chunk（batch + 队列，可并行）
3. 用同一评测集对比 v1 vs v2：Recall@K、MRR、NDCG、P95 延迟
4. 小流量灰度（5% → 20% → 100%），监控无答案率与引用采纳率
5. 保留 v1 索引一段时间，支持一键回滚
6. 全量切换后下线 v1，记录 embedding_model_version
```

**禁止：** 新旧模型向量写入同一索引混搜——向量空间不一致，相似度无意义。

#### 版本字段建议

`document_id + chunk_index + document_version + embedding_model_version + chunk_hash`  
排障时可精确归因：是文档变了、切分变了、还是模型/索引变了。

---

### 8. 容量规划（完整估算）

**裸向量内存（float32）：**

```text
原始向量大小 ≈ chunk 数量 × 维度 × 4 字节

例：1,000,000 chunk × 1536 维 × 4 B ≈ 6.1 GB（仅向量数组）
```

**实际内存远大于裸向量：**

| 组成部分 | 说明 |
| --- | --- |
| HNSW 图边 | 常为数倍向量体积，与 M 正相关 |
| metadata | title、acl、text 摘要等 |
| 副本/高可用 | 主从、多 AZ 复制 |
| 双版本并存 | 升级灰度期 v1+v2 同时存在 |
| 构建临时空间 | 重建索引峰值 |

**面试完整答法：** 不只报 6GB，要说明 HNSW 图、metadata、副本、双索引灰度后 **10～30GB 量级很常见**，需结合云厂商实例规格与扩容策略。

**磁盘 vs 内存：** 部分向量库支持磁盘索引（如 Milvus 磁盘版、pgvector + 合适配置），用延迟换成本，适合冷数据或超大库。

---

### 9. 召回评估（完整方法论）

**不要只用最终答案准确率判断 Embedding**——答案还受 rerank、Prompt、生成影响。

#### 评测集构建

每条样本至少包含：

- `query`：用户问题（含多轮改写版更佳）
- `gold_chunk_ids` 或 `gold_passages`：标准相关片段
- `标签`：正常 / 边界 / 歧义 / 专有名词 / 短 query

#### 核心指标（须会解释）

| 指标 | 含义 | 用途 |
| --- | --- | --- |
| **Recall@K** | 金标准是否出现在 top K | 初召是否「捞得到」 |
| **Precision@K** | top K 里有多少真正相关 | 噪声多不多 |
| **MRR** | 第一个相关结果排名的倒数均值 | 排序质量 |
| **NDCG** | 考虑多级相关度的排序质量 | 多级标注场景 |

```python
def recall_at_k(retrieved_ids: list[str], gold_ids: set[str], k: int) -> float:
    """中文注释：金标准任一命中 top_k 即算召回成功"""
    top = set(retrieved_ids[:k])
    return 1.0 if top & gold_ids else 0.0
```

#### 对照实验（定位问题标准流程）

1. **固定 Embedding，换 chunk 策略** → 改的是切分/解析
2. **固定 chunk，换 Embedding 模型** → 改的是语义空间
3. **金标准不在任何 chunk** → 文档缺失、PDF 解析、切分断了上下文
4. **在 chunk 但排名靠后** → 向量模型、ANN 参数、缺混合检索、缺 rerank
5. **召回好但答案错** → 往上查 rerank、上下文压缩、生成，不是 Embedding 层单独背锅

---

### 10. 短 query、专有名词与难例（完整应对）

| 难例 | 原因 | 手段 |
| --- | --- | --- |
| **「怎么办」** | 语义空泛，向量不确定 | 多轮改写、补全指代 |
| **「E12345 错误」** | 错误码需精确匹配 | BM25 加权、metadata 过滤产品 |
| **SKU / API 名** | 稀有 token embedding 弱 | 混合检索、词典扩展 |
| **中英混合** | 模型对混合语料训练不足 | 选多语言模型或领域微调 |
| **极短词** | 向量信号弱 | 入口默认 kb 范围、引导用户补全 |

不能单靠「把 top_k 调大」——噪声会拖垮 rerank 和 Prompt。

---

### 11. 离线索引构建与在线检索（完整工程链路）

#### 离线批处理

```python
async def index_document(doc, embedder, vector_store, chunker):
    chunks = chunker.split(doc.content, max_tokens=512, overlap=64)
    texts = [c.text for c in chunks]
    # 中文注释：document 批量 embed 提高吞吐
    vectors = await embedder.encode_batch(texts, batch_size=32)
    records = [
        {
            "chunk_id": f"{doc.id}#{i}",
            "document_id": doc.id,
            "text": c.text,
            "embedding": vec,
            "metadata": {
                "tenant_id": doc.tenant_id,
                "acl": doc.acl,
                "document_version": doc.version,
                "embedding_model_version": embedder.version,
            },
        }
        for i, (c, vec) in enumerate(zip(chunks, vectors))
    ]
    await vector_store.upsert(records)
```

要点：**batch embed**、失败重试、幂等 upsert、索引构建与查询分离（bulk 后再 reload/load index）。

#### 在线检索 SLA 分解

```text
总延迟 ≈ query_embed + ann_search + filter_merge + rerank + 组装
```

典型优化：query embedding 缓存（相同 query）、ANN efSearch 与 Recall 折中、rerank 限制候选数、向量库与应用同地域部署。

---

### 12. 线上监控与排障（完整清单）

| 监控项 | 说明 | 异常含义 |
| --- | --- | --- |
| query embedding 延迟/失败率 | API 或自托管健康 | 模型服务瓶颈 |
| 向量检索 P95 | ANN 性能 | efSearch/M/数据量问题 |
| 初召候选数 | top_k 实际返回条数 | 过滤过严、索引空 |
| 过滤后空结果率 | 权限/kb 过滤后无命中 | acl 配置、数据缺失 |
| 最高分分布 | 相似度整体偏低 | 模型不匹配、query 太难 |
| rerank 延迟 | 精排瓶颈 | 候选过多 |
| 无答案率 / 拒答率 | 业务体感 | 召回或生成问题 |
| 用户点击引用 chunk | 采纳率 | 召回相关性真实反馈 |

**排障口诀：** 没召到 → chunk/embedding/索引/过滤；召得慢 → ANN 参数/规模/硬件；召得错 → 混合检索/rerank；召到但答错 → 生成与上下文，别只怪 Embedding。

---

## 高频面试问题与标准答案

### 1. Embedding 和 Reranker 有什么区别？

**标准答案：**  
Embedding 是 Bi-Encoder，query 和 document 分开编码，用向量距离做第一阶段**粗召回**，适合在百万级 chunk 里快速找候选，追求高 Recall 和低延迟。Reranker 一般是 Cross-Encoder，把 query 和 passage 拼在一起打分，算力随候选数线性涨，所以只用在几十条候选上做**精排**，追求排序质量和 Precision。生产上常见「向量召 50～100 条，rerank 留 5～10 条进 Prompt」，两阶段分工不同，不能互相替代。

### 2. 向量很像，为什么答案还是错的？

**标准答案：**  
我会分层排查，不会一句「模型幻觉」带过。第一，chunk 语义相似但**不含答案**，比如「退款条件」和「到账时间」向量近，但只有后者有「7 个工作日」。第二，query 太短或指代不明，向量信号弱。第三，专有名词、错误码 embedding 不准，需要 BM25 补。第四，top_k 太小或 ANN 参数太激进，金标准 chunk 没进候选。第五，metadata 过滤误杀或过滤后池子太小。第六，rerank 或上下文裁剪把对的 chunk 挤掉了。所以要对照：金标准在不在 chunk 里、在不在 top_k 里、进没进最终 Prompt。

### 3. HNSW 为什么吃内存？

**标准答案：**  
HNSW 不只存向量本体，还要存**分层导航图**的邻居边，参数 M 越大每个节点连接越多，召回越好但内存越高。再加上 metadata、副本、构建时的临时结构，整体往往是裸向量体积的**数倍到十几倍**。所以容量规划不能只用「chunk 数 × 维度 × 4 字节」，还要把图索引和 HA 算进去。

### 4. top_k 是不是越大越好？

**标准答案：**  
不是。top_k 太小容易漏召回；太大则噪声进 rerank 和 Prompt，延迟、token 成本上升，还可能引入无关上下文导致幻觉。正确做法是用评测集画 Recall@K 曲线找拐点，再结合线上 P95 延迟和 rerank 成本定默认值。还要区分：top_k 是初召条数，ANN 的 efSearch 不够时，加大 top_k 也捞不到图搜索外的向量。

### 5. Embedding 模型升级要注意什么？

**标准答案：**  
核心是新模型和旧模型的向量空间**一般不兼容**，不能混在同一索引里搜。要新建 collection 全量重嵌，用同一评测集对比 Recall@K、MRR 和延迟，小流量灰度，保留旧索引支持回滚。metadata 里记录 `embedding_model_version`，query 和 document 必须同一版本。升级窗口要预估离线 embed 时间和双索引并存的双倍存储。

### 6. metadata 过滤为什么必须在检索层做，不能只在应用层？

**标准答案：**  
如果先向量召回再应用层过滤，无权限的 chunk 可能已经写进检索日志、rerank 输入甚至 LLM 上下文，有合规和泄露风险。租户、acl、deleted 这类过滤必须在向量库或检索服务层执行。业务类排序像新鲜度可以 rerank 后置，但安全类必须前置。

### 7. 向量库和 Elasticsearch 是替代关系吗？

**标准答案：**  
不完全是。向量库（Milvus、Qdrant、pgvector 等）擅语义 ANN；Elasticsearch/OpenSearch 擅 BM25、复杂过滤、聚合和成熟运维。很多企业是**向量 + ES 混合检索**，再 rerank 融合。pgvector 适合中小规模、希望 SQL 和向量一体的场景；超大规模可能专用向量库 + ES 关键词双写。

### 8. 用户 query 很短怎么办？

**标准答案：**  
短 query 向量语义信号弱，不能单靠相似度。可以结合多轮对话改写补全指代；用 BM25 抓关键词；对错误码、SKU 做精确匹配或词典；入口默认限定知识库范围减少搜索空间；必要时引导用户补全信息。HyDE、Multi-Query 等也能扩充 query 表达，但要权衡多一次 LLM 调用的成本。

### 9. 怎么判断是 Embedding 问题还是 chunk 切分问题？

**标准答案：**  
做对照实验：固定 Embedding 模型换 chunk 策略，看 Recall 是否显著变化；再固定 chunk 换模型。然后人工看 badcase：如果**正确答案根本不在任何 chunk 里**，是 PDF 解析、切分粒度或文档缺失问题；如果在某个 chunk 里但排名靠后，才是向量模型、ANN 参数、混合检索或 rerank 问题。这个区分决定优化方向，避免盲目换模型。

### 10. HNSW 参数怎么调？

**标准答案：**  
先用 Flat 或很高 efSearch 当 Recall 上限基准。然后在评测集上扫 efSearch，看 Recall 和 P95 延迟的拐点；M 和 efConstruction 影响索引质量和构建时间，M 大召回好但内存高。线上不要凭感觉，要**评测集 + 延迟 + 内存**一起看。候选不足时先查过滤是否过严，再加大 efSearch，而不是只加 top_k。

### 11. 为什么要记录 embedding_model_version？

**标准答案：**  
召回异常时要快速归因：是文档更新了、切分策略变了，还是 embedding 模型或索引版本变了。灰度期间可能 v1、v2 双索引并存，版本字段支持按租户/流量路由和一键回滚。也方便审计某次回答用的是哪套向量空间，避免排查时混用不同版本的数据。

### 12. Cosine、内积、L2 能混用吗？

**标准答案：**  
不能随意混。要和 Embedding 模型训练目标和向量库索引配置一致。很多模型输出已 L2 归一化，此时 cosine 和内积排序等价；若未归一化，内积会受向量模长影响，长文本可能占优。换度量或换模型后需要重建索引，不能拿旧向量直接换相似度函数。

### 13. 召回层应该监控哪些指标？

**标准答案：**  
我会看 query embedding 延迟和失败率、向量检索 P95、初召候选数、过滤后空结果率、相似度最高分分布、rerank 耗时、无答案率和用户点击引用的 chunk 采纳率。这样能区分是「没召到」「召得慢」「召得错」还是「过滤太严」。结合 trace_id 把一次请求的初召列表和最终引用对齐，badcase 归因最快。

### 14. 如何做混合检索的分数融合？

**标准答案：**  
向量分和 BM25 分量纲不同，不能直接相加 unless 做过归一化。常见做法：RRF（Reciprocal Rank Fusion，按排名融合，不依赖绝对分数）；或 min-max 归一化后加权 `α * vector + (1-α) * bm25`，α 用评测集调。合并时按 chunk_id 去重，避免同一 passage 占两个名额。最终仍建议过 rerank，融合只是粗召阶段的互补。

---

## 面试回答加分点

1. **系统视角**：把 Embedding 层讲成「召回 + 权限 + 索引性能 + 版本治理 + 评估闭环」，不是一次 `similarity_search` API 调用。
2. **HNSW 三角**：能口述 M / efConstruction / efSearch 对 Recall、延迟、内存的影响，并给出「Flat 基准 → 扫 efSearch → 调 M」的调参顺序。
3. **混合检索标配**：向量管语义，BM25 管错误码/SKU/API 名，rerank 管精排——三层结构体现企业实践。
4. **版本与灰度**：模型升级 = 新 namespace + 全量重嵌 + 评测 + 灰度 + 回滚；强调 `embedding_model_version` 字段。
5. **对照实验定位**：金标准不在 chunk → 切分/解析；在 chunk 但排后面 → 向量/索引/rerank——体现工程排障能力。
6. **容量估算完整**：裸向量 + HNSW 图 + metadata + 副本 + 双版本灰度，避免只报「6GB」显得没做过生产。
7. **安全前置**：权限过滤在检索层，不是应用层事后 discard——合规意识。
8. **一分钟口述模板**：同一模型版本 encode query/doc → ANN 初召 + metadata 前置过滤 → 混合 BM25 → rerank → 评测用 Recall@K 对照 chunk 与模型 → 升级全量重嵌灰度回滚。
