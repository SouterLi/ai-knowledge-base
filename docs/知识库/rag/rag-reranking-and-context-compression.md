# AI 应用开发面试主题：RAG 重排序与上下文压缩

## 核心概念

RAG 在线链路里，**召回之后、生成之前** 还有一层质量控制：从候选证据里挑出真正能回答问题、且在 token 预算内可溯源的内容。

**两阶段检索**：

```text
用户问题 -> 多路召回(向量/BM25/多查询) -> 合并去重 -> Rerank 精排
  -> 多样性(MMR等) -> 上下文压缩/组装 -> LLM 生成
```

- **初召（Recall）**：宁可多拿候选，追求覆盖面与速度。  
- **重排（Rerank）**：追求精度与可回答性，把含答案的片段排到前面。  
- **上下文压缩**：在有限 token 内保留证据，不是简单截断——压缩后仍要能 **回到原文 span**。

> 向量 top_k 像粗筛；reranker 像精筛；压缩负责「塞得下且塞得对」。

| Rerank 类型 | 特点 | 典型场景 |
| --- | --- | --- |
| Cross-Encoder | query+passage 联合编码，精度高 | 企业知识库主链路 |
| Bi-Encoder / 向量分 | 快，细粒度弱 | 候选极大、延迟敏感 |
| LLM Rerank | 灵活、贵、不稳定 | 小候选、难例、离线评测 |
| 规则 / LTR | 标题、新鲜度、权威、点击 | 强业务信号 |

---

## 核心知识点

### 1. 为何不直接把向量 top_k 给 LLM？

1. **相似 ≠ 可回答**（问到账时间却召回「如何申请退款」）。  
2. 局部关键词相同但对象不同（企业版 vs 个人版）。  
3. 多路召回 **重复** 浪费窗口。  
4. ANN 优化的是距离，不是最终答题质量。  
5. 长上下文 **≠** 高信噪比，噪声会增加错误引用。

### 2. Cross-Encoder 原理与规模

```text
[CLS] 用户问题 [SEP] 候选片段 [SEP] -> relevance_score
```

query 与 passage token 可交叉注意力，对否定、限定条件更敏感。  
成本随候选数线性增长 → 常见：初召 50–200，rerank 20–100，进 Prompt 3–10 段。

### 3. 分数融合（勿迷信单一分）

```python
def compute_final_score(c: Candidate) -> float:
    # 不同分数量纲要先归一化；权限不能靠降权代替过滤
    return (
        0.60 * norm(c.rerank_score)
        + 0.15 * norm(c.vector_score)
        + 0.10 * norm(c.bm25_score)
        + 0.07 * norm(c.metadata.get("title_match_score", 0))
        + 0.05 * norm(c.metadata.get("freshness_score", 0))
        + 0.03 * norm(c.metadata.get("authority_score", 0))
    )
```

权重用离线 NDCG、线上采纳率迭代，不要长期拍脑袋固定。

### 4. 候选合并与去重

按 `chunk_id`、文本 hash、**文档版本** 去重；相邻 chunk 可合并成更完整段落。  
顺序：各路保留来源分 → 合并 → 再排序；不要过早丢掉 BM25/向量分信号。

### 5. MMR（相关性与多样性）

```text
score(c) = λ * rel(query,c) - (1-λ) * max_sim(c, selected)
```

适合对比、多来源总结；**不适合** 单一精确事实（只要最权威一条）。λ 大更偏相关，小更偏多样。

### 6. 上下文压缩策略

| 策略 | 优点 | 风险 |
| --- | --- | --- |
| 句子窗口 | 低成本、保原文 | 表格/跨段答案差 |
| LLM 摘要 | 压 token 明显 | 丢数字/例外条款，易摘要幻觉 |
| 结构化抽取 | 政策/配置稳定 | 需 schema |

**高风险场景**（法务、合同、金额）：优先抽取式 + 保留引用 span，不用纯生成摘要当唯一证据。

### 7. Token 预算分配

```text
总窗口 = 系统提示 + 用户问题 + 历史(可摘要) + 检索证据 + 输出预留
```

先预留输出 token；每文档设占比上限；低分候选设阈值，**宁可拒答不塞噪声**。

### 8. 工程职责拆分

```python
def build_rag_context(query: str, token_budget: int) -> list[Candidate]:
    candidates = merge_dedupe(
        vector_retrieve(query, top_k=80),
        bm25_retrieve(query, top_k=80),
    )
    candidates = filter_by_acl(candidates)  # ACL 必须在 rerank 前
    candidates = rerank(query, candidates[:100])
    candidates = diversify_with_mmr(query, candidates, limit=12)
    return compress_candidates(query, candidates, token_budget)
```

### 9. 排障顺序（调优别先换大模型）

1. 金标准是否在库；2. 初召 top_100 是否含正确 chunk；3. rerank 后是否进 top_5；4. 压缩是否删掉数字/条件；5. Prompt 是否要求引用；6. 再查生成模型。

### 10. 评估指标

排序：Recall@K、MRR、NDCG、Context Precision/Recall。  
生成：Faithfulness、Citation Accuracy、拒答率；对比压缩前后 token 与忠实度是否 trade-off 失控。

---

## 高频面试问题与标准答案

### 1. Reranker 和 Embedding 检索区别？

**标准答案：**  
Embedding 是百万级粗召，双塔编码快但交互弱。Reranker 对少量候选做 query-passage 精排，更能判断「能不能答这道题」。我会说粗召负责「别漏」，精排负责「别脏」。

### 2. Cross-Encoder 为什么通常更准？

**标准答案：**  
因为它同时看 query 和 passage 的 token 交互，能抓住限定词、否定和实体对应；向量检索把信息压进两个独立向量，细粒度判断弱。代价是慢，所以只 rerank 初召后的子集。

### 3. rerank 候选数怎么定？

**标准答案：**  
看延迟预算和初召质量。常见初召上百、rerank 几十、最终进 Prompt 个位数。若金标准经常不在 rerank 输入里，先加大初召或改混合检索；若在输入里但排后面，调 reranker 或融合权重。

### 4. 多路召回后怎么合并排序？

**标准答案：**  
保留每路的 rank 和原始分，chunk_id 去重，分数归一化后融合 rerank、向量、BM25、标题命中、新鲜度等。权限过滤在排序前完成，无权限文档不能进候选再降权。

### 5. MMR 解决什么问题？

**标准答案：**  
防止 top_k 全是同一文档的相似片段。在相关性和与已选片段的差异性之间折中。适合对比、总结类；精确事实题通常不需要 MMR，只要最高分那条权威证据。

### 6. 上下文压缩和摘要有什么区别？

**标准答案：**  
压缩是目标，摘要只是手段之一。还可以句子窗口、表格行抽取、去重合并。摘要可能改写字面意思，关键业务要保留原文引用；我会说压缩的验收标准是 Faithfulness 和引用准确率不能掉。

### 7. 长上下文模型还需要 rerank 吗？

**标准答案：**  
需要。长窗口解决「装得下」，不解决「该装什么」。噪声多会增加成本和错误引用，重要证据仍应靠前放；rerank 管信噪比和顺序。

### 8. 怎么判断是召回、排序还是生成问题？

**标准答案：**  
看正确 chunk 在不在索引；在不在初召 top_100；rerank 后是否进 top_5；最终 Prompt 里关键句还在不在；都有但答错才查模型和 Prompt。这套分层说法面试官一般认可。

### 9. rerank 分数很低怎么办？

**标准答案：**  
设阈值拒答或请用户澄清；可扩大召回、开 BM25、多查询改写；高风险转人工。不要在低置信证据上硬生成，幻觉和合规风险都高。

### 10. 怎么评估压缩是否伤效果？

**标准答案：**  
对比压缩前后：答案是否仍可由上下文推出；金额日期版本号是否保留；引用是否仍指向原文；token/延迟是否下降；Faithfulness 和 Citation Accuracy 是否变差。变短但引用错了说明压缩策略有问题。

### 11. 结果重复很多怎么优化？

**标准答案：**  
索引层减 overlap、清重复文档、只保留当前版本；召回层 chunk_id/hash 去重；排序层 MMR 或每文档限入选条数；相邻 chunk 可合并而非简单删。

### 12. LLM Rerank 适合生产主链路吗？

**标准答案：**  
一般不当作默认主路径，成本高、延迟高、输出不稳定。更适合小候选、复杂判断或离线标注。生产主链路用专用 Cross-Encoder，LLM 作难例增强或评测辅助。

### 13. 如何设计高质量检索排序链路？（综合题）

**标准答案：**  
混合召回+权限前置；多路合并去重保留来源分；Cross-Encoder rerank 并融合业务信号；对比类用 MMR；按 token 预算压缩且保留引用；低分拒答；用 NDCG、Context Precision、Faithfulness 和线上坏 Case 回归调参。

### 14. 用户说「引用了无关文档」怎么排查？

**标准答案：**  
先看最终 Prompt 里是否真的注入了该文档；追来源是向量、BM25 还是多查询；看 rerank 分和融合分是否异常；查 metadata 过滤和版本；看压缩是否把片段裁成误导性短句；加入回归评测集。

### 15. 低延迟与高质量如何权衡？

**标准答案：**  
限制 rerank 候选、批处理、缓存热门 query；简单事实走快链路，复杂分析走高质量链路；rerank 超时可有降级（混合分排序），但金融法务等高风险场景不能无脑降级生成。

---

## 面试回答加分点

- 用 **「初召要覆盖、精排要准、压缩要保真」** 三句话概括全链路。
- 强调 **权限在 rerank 前**，分数融合不能替代 ACL。
- 能说 **MMR、摘要、句子窗口** 的适用与风险，体现不是只会调 top_k。
- 长上下文观点清醒：**容量 ≠ 证据质量**。
- 排障有 **分层 checklist**，显得做过线上 badcase。
- 评估提到 **Context Precision、Citation Accuracy**，连接 RAGAS 类指标体系。
