# AI 应用开发面试主题：RAG 重排序与上下文压缩

## 主题选择记录

- 本次主题：RAG 重排序与上下文压缩
- 所属分类：AI 应用开发 / RAG
- 已回顾仓库主题：RAG 系统设计与评估、RAG 文档摄取与切分、Embedding 与向量索引工程、GraphRAG、Agent、Prompt 工程、LLMOps、安全、多租户、上下文工程、流式架构、多模态、MCP、模型网关、测试、发布治理、实时语音 Agent 等
- 不重复原因：已有 RAG 文档会提到 reranker，但没有单独展开 **候选召回合并、重排序模型、分数融合、多样性控制、上下文压缩、token 预算、证据保真和评估调优**。本主题聚焦 RAG 在线检索链路中“召回之后、生成之前”的质量控制层。
- 适用岗位：AI 应用开发工程师、RAG 工程师、LLM 应用工程师、搜索推荐工程师、AI 平台工程师

---

## 一、为什么这是高频考点

很多 RAG 系统上线后，问题不在于“有没有检索”，而在于：

- 初召 top_k 里混入大量相似但答非所问的 chunk。
- 正确 chunk 被召回了，但排在第 20 名之后，没有进入 prompt。
- 多路召回结果重复，浪费上下文窗口。
- 单个 chunk 太长，塞进 prompt 后挤掉了更关键的证据。
- 长上下文模型可容纳更多 token，但噪声更多，模型更容易引用错误片段。
- 只看向量相似度，忽略标题、章节、权限、新鲜度、点击反馈等业务信号。

面试官问重排序与上下文压缩，通常是在考察你是否理解：**RAG 不是把召回结果直接塞给大模型，而是要在有限上下文预算内挑出最相关、最完整、最低噪声、可溯源的证据。**

一句话回答：

> 重排序负责从候选证据中选出真正能回答问题的内容；上下文压缩负责在不破坏事实依据的前提下，把证据裁剪成适合 LLM 使用的输入。

---

## 二、核心概念

### 1. 两阶段检索：Recall 与 Rerank

生产级 RAG 常用两阶段甚至多阶段检索：

```text
用户问题
  -> Query Rewrite / 多查询生成
  -> 向量召回、BM25 召回、metadata 过滤、多粒度召回
  -> 候选合并与去重
  -> Reranker 重排序
  -> 多样性与业务规则调整
  -> 上下文压缩与组装
  -> LLM 生成答案
```

第一阶段召回关注 **召回率、速度和覆盖面**，宁可多拿一些候选。

第二阶段重排序关注 **精度、顺序和可回答性**，把最有用的证据排到前面。

面试表达：

> 向量检索像粗筛，用近似最近邻从海量 chunk 中快速找候选；reranker 像精筛，对 query 与候选内容做更细粒度的相关性判断。

### 2. Reranker 是什么

Reranker 输入通常是：

- query：用户问题或改写后的查询。
- candidate passages：初召返回的候选 chunk。
- metadata：标题、路径、时间、权限、来源、文档类型等。

输出通常是：

- 每个候选的相关性分数。
- 排序后的候选列表。
- 可选的拒绝信号，例如分数低于阈值时触发“不知道”或扩大召回。

典型重排方式：

| 类型 | 原理 | 优点 | 缺点 | 适用场景 |
| --- | --- | --- | --- | --- |
| Cross-Encoder Reranker | 将 query 和候选文本拼接输入模型，输出相关性分数 | 精度高，排序质量稳定 | 延迟高，候选数不能太大 | 企业知识库、客服问答、政策检索 |
| Bi-Encoder 相似度重排 | 使用向量相似度或轻量模型重新打分 | 快，成本低 | 对细粒度匹配能力弱 | 候选量大、延迟敏感 |
| LLM Rerank | 让 LLM 判断候选是否能回答问题 | 灵活，可解释 | 成本高，稳定性差，延迟高 | 小候选集、复杂推理、人工审核辅助 |
| 规则重排 | 根据标题匹配、时间、文档权威性等规则加权 | 可控，业务解释性强 | 难覆盖语义相关性 | 强业务规则场景 |
| Learning-to-Rank | 用点击、标注、转化数据训练排序模型 | 可持续优化 | 需要高质量反馈数据 | 搜索流量充足的平台型系统 |

### 3. 上下文压缩是什么

上下文压缩不是简单截断文本，而是在有限 token 预算内保留可回答问题的证据。

常见做法：

- 片段级裁剪：只保留命中的句子、段落和相邻上下文。
- 摘要压缩：用模型把长证据压缩成短证据。
- 结构化抽取：提取表格行、字段、条款、数字、时间、实体关系。
- 去重合并：合并重复或高度相似的 chunk。
- 分层上下文：先放核心证据，再放背景说明，再放引用元数据。

关键原则：

> 压缩后仍然要能追溯到原文，不能把模型生成的摘要当成不可验证的事实来源。

---

## 三、核心知识点

### 1. 为什么不能直接把向量检索 top_k 给 LLM

向量检索 top_k 的问题：

1. **相似不等于可回答**
   用户问“退款多久到账”，向量召回可能返回大量“退款申请条件”，但不包含到账时间。

2. **局部语义容易误判**
   chunk 中有几个关键词相同，但上下文对象不同，例如“企业版”和“个人版”规则不同。

3. **多路召回会产生重复**
   BM25、向量、多查询召回可能返回同一段或相邻段。

4. **排序目标不同**
   ANN 检索优化的是向量距离，不一定优化最终回答质量。

5. **上下文窗口有限**
   即使模型支持长上下文，也不代表应该把噪声都塞进去。

面试回答可以这样说：

> 初召的目标是高召回，rerank 的目标是高精度。直接取向量 top_k 会把“语义相似但不能回答问题”的片段带入 prompt，增加幻觉和引用错误风险。

### 2. Cross-Encoder Reranker 的工作方式

Cross-Encoder 会把 query 和 passage 作为一对输入：

```text
[CLS] 用户问题 [SEP] 候选片段 [SEP] -> relevance_score
```

它比 Embedding 检索更精细，因为模型可以在 query 和 passage 的 token 之间做交叉注意力。

优点：

- 更擅长判断“候选是否真的回答问题”。
- 对精确字段、否定关系、限定条件更敏感。
- 对多路召回后的候选精排效果好。

缺点：

- 每个 query-candidate 都要跑一次模型，成本随候选数线性增长。
- 输入长度有限，长 chunk 需要截断或先裁剪。
- 延迟较高，需要批处理、并发或候选数控制。

常见候选规模：

- 初召：50-200 个 chunk。
- rerank 输入：20-100 个 chunk。
- 进入 prompt：3-10 个片段，具体取决于 token 预算和问题复杂度。

### 3. 分数融合：不要只迷信一个分数

生产环境里，最终排序通常不是单一 reranker 分数，而是多信号融合：

```text
final_score =
  0.55 * reranker_score
  + 0.20 * normalized_vector_score
  + 0.10 * bm25_score
  + 0.08 * title_match_score
  + 0.05 * freshness_score
  + 0.02 * authority_score
```

常见信号：

| 信号 | 作用 |
| --- | --- |
| reranker_score | 判断 query 与候选是否语义相关 |
| vector_score | 保留语义相似度信息 |
| bm25_score | 强化关键词、编号、错误码、专有名词 |
| title_match_score | 标题、章节命中通常代表上下文更权威 |
| freshness_score | 政策、价格、接口文档等需要新鲜度 |
| authority_score | 官方文档、已审核文档优先 |
| feedback_score | 点击、采纳、人工标注等线上反馈 |
| permission_score | 不用于提权，只用于已授权候选内的排序 |

注意：权限过滤必须在排序前完成，不能通过降低分数来代替权限控制。

### 4. 候选合并与去重

多路召回后，候选集合经常包含重复内容：

- 同一个 chunk 被向量和 BM25 同时召回。
- 相邻 chunk 内容高度重叠。
- 同一文档不同版本同时存在。
- 多查询改写召回了同一批结果。

去重策略：

1. **稳定 ID 去重**：按 `chunk_id`、`document_id + span` 去重。
2. **文本指纹去重**：对规范化文本做 hash。
3. **相似度去重**：对高度相似 chunk 只保留分数最高的。
4. **版本去重**：同一文档只保留最新可见版本。
5. **相邻合并**：同一文档连续 chunk 可以合并成更完整段落。

面试易错点：

> 去重不是越早越好。通常先在同一路召回内去重，再合并多路结果，保留每个候选的来源信号，最后再做排序和相邻合并。

### 5. MMR：相关性与多样性的平衡

MMR（Maximal Marginal Relevance）用于避免 top_k 全是同一类相似内容。

目标：

- 选中的片段要和 query 相关。
- 新选片段要和已选片段不太重复。

简化公式：

```text
score(candidate) =
  lambda * relevance(query, candidate)
  - (1 - lambda) * max_similarity(candidate, selected)
```

`lambda` 越大，越重视相关性；越小，越重视多样性。

适用场景：

- 问题需要多个方面的证据，例如“对比企业版和个人版限制”。
- 召回结果高度重复。
- 需要覆盖不同章节、不同文档或不同观点。

不适用场景：

- 用户问一个精确事实，只需要最权威的一条证据。
- 候选质量很差，多样性会放大噪声。

### 6. 上下文压缩的常见策略

#### 策略一：句子窗口裁剪

先定位 chunk 中与 query 最相关的句子，再保留前后少量句子。

优点：

- 成本低，可解释。
- 保留原文，不容易引入摘要幻觉。

缺点：

- 对表格、列表、跨段落答案不友好。
- 句子切分质量影响效果。

#### 策略二：LLM 摘要压缩

让模型把候选证据压缩成短上下文。

适合：

- 单个候选很长。
- 证据分散在多个段落。
- 需要把表格、FAQ、流程说明改写成更紧凑的形式。

风险：

- 摘要可能丢数字、条件、例外条款。
- 摘要可能引入原文没有的信息。
- 如果不保留引用，后续答案难以溯源。

工程建议：

- 摘要输出必须带原文引用 span。
- 对金额、日期、阈值、版本号等字段尽量抽取，不要自由改写。
- 重要场景优先使用抽取式压缩，而不是生成式摘要。

#### 策略三：结构化抽取

把证据转成结构化 JSON，适合政策、配置、接口、表格类问答。

```json
{
  "question": "企业版退款多久到账？",
  "evidence": [
    {
      "doc_id": "refund-policy-v3",
      "title": "企业版退款政策",
      "matched_span": "审核通过后，退款将在 5-7 个工作日内原路返回。",
      "fields": {
        "product": "企业版",
        "condition": "审核通过",
        "refund_time": "5-7 个工作日",
        "method": "原路返回"
      }
    }
  ]
}
```

优点：

- 对下游生成更稳定。
- 便于检查字段缺失。
- 适合做引用、审计和评估。

缺点：

- 需要定义 schema。
- 对开放领域长文档不一定通用。

### 7. Token 预算怎么分配

上下文组装不是“能塞多少塞多少”，而是要明确预算：

```text
总上下文窗口
  = 系统提示词
  + 开发者约束
  + 用户问题
  + 对话历史
  + 检索证据
  + 工具结果
  + 输出预留 token
```

常见策略：

- 先预留输出 token，避免答案被截断。
- 对话历史先摘要，再放近期原文。
- 检索证据按分数、覆盖面和引用价值排序。
- 每个文档设置最大 token 占比，避免单一文档霸占上下文。
- 对低分候选设置阈值，宁可回答“不知道”，不要塞噪声。

面试中可给一个经验回答：

> 我会把证据预算当成显式资源管理：先按任务预留系统提示、历史和输出空间，再根据 query 类型决定证据数量。事实问答通常少量高置信证据即可；对比、总结类问题需要覆盖多个来源，但要做去重和压缩。

### 8. 什么时候不应该压缩

不是所有场景都适合压缩。

不建议压缩的场景：

- 法律、合同、合规条款，需要保留原文。
- 数字、公式、代码、配置项容易被改错。
- 用户明确要求引用原文。
- 后续需要把答案和原文逐字核对。

可以压缩的场景：

- 背景解释很长，但答案只依赖少量句子。
- 多篇文档需要汇总，单篇原文都太长。
- 文档格式噪声多，需要清理无关导航、页眉页脚。

---

## 四、工程实现示例

### 1. 检索、重排、压缩的接口拆分

```python
from dataclasses import dataclass
from typing import Any


@dataclass
class Candidate:
    chunk_id: str
    document_id: str
    text: str
    metadata: dict[str, Any]
    vector_score: float = 0.0
    bm25_score: float = 0.0
    rerank_score: float = 0.0
    final_score: float = 0.0


def build_rag_context(query: str, token_budget: int) -> list[Candidate]:
    vector_hits = vector_retrieve(query, top_k=80)
    keyword_hits = bm25_retrieve(query, top_k=80)

    # 合并多路召回结果时保留来源分数，后续排序需要这些信号
    candidates = merge_and_deduplicate(vector_hits, keyword_hits)

    # 先用轻量规则过滤明显不可用的候选，再把有限候选交给 reranker
    candidates = filter_by_acl_and_metadata(candidates)
    candidates = rerank(query, candidates[:100])
    candidates = diversify_with_mmr(query, candidates, limit=12)
    candidates = compress_candidates(query, candidates, token_budget=token_budget)

    return candidates
```

这里的重点不是函数名，而是职责拆分：

- `retrieve`：负责召回。
- `merge_and_deduplicate`：负责候选归并。
- `filter_by_acl_and_metadata`：负责权限和业务过滤。
- `rerank`：负责精排。
- `diversify_with_mmr`：负责减少重复证据。
- `compress_candidates`：负责上下文预算控制。

### 2. 分数融合示例

```python
def normalize(value: float, minimum: float, maximum: float) -> float:
    if maximum <= minimum:
        return 0.0
    return (value - minimum) / (maximum - minimum)


def compute_final_score(candidate: Candidate) -> float:
    normalized_rerank_score = normalize(candidate.rerank_score, 0.0, 1.0)
    normalized_vector_score = normalize(candidate.vector_score, 0.0, 1.0)
    normalized_bm25_score = normalize(candidate.bm25_score, 0.0, 30.0)
    normalized_freshness_score = normalize(
        candidate.metadata.get("freshness_score", 0.0),
        0.0,
        1.0,
    )
    normalized_title_match_score = normalize(
        candidate.metadata.get("title_match_score", 0.0),
        0.0,
        1.0,
    )
    normalized_authority_score = normalize(
        candidate.metadata.get("authority_score", 0.0),
        0.0,
        1.0,
    )

    # 权限不能靠打分处理，候选进入这里前必须已经完成 ACL 过滤
    return (
        0.60 * normalized_rerank_score
        + 0.15 * normalized_vector_score
        + 0.10 * normalized_bm25_score
        + 0.07 * normalized_title_match_score
        + 0.05 * normalized_freshness_score
        + 0.03 * normalized_authority_score
    )
```

面试要点：

- 不同分数尺度不同，融合前要归一化。
- 权重不要拍脑袋长期固定，应通过离线评测和线上反馈调。
- 对强规则场景，可以先 hard filter，再 soft ranking。

### 3. 上下文组装示例

```python
def assemble_context(candidates: list[Candidate], token_budget: int) -> str:
    blocks: list[str] = []
    used_tokens = 0

    for index, candidate in enumerate(candidates, start=1):
        source = candidate.metadata.get("source", "unknown")
        title = candidate.metadata.get("title", "untitled")
        text = candidate.text.strip()

        block = f"[证据 {index}]\n来源：{source}\n标题：{title}\n内容：{text}\n"
        block_tokens = estimate_tokens(block)

        if used_tokens + block_tokens > token_budget:
            continue

        blocks.append(block)
        used_tokens += block_tokens

    return "\n".join(blocks)
```

好的上下文不仅有内容，还应该有：

- 来源信息：文档、URL、页码、章节。
- 证据编号：方便 LLM 引用。
- 明确边界：避免不同文档内容混在一起。
- 排序顺序：高置信证据放前面。

---

## 五、常见架构模式

### 1. 低延迟问答链路

适合客服、助手、交互式问答：

```text
query -> hybrid recall top 50 -> cross-encoder rerank top 20 -> select top 5 -> sentence window compression -> answer
```

特点：

- 控制候选数和模型调用次数。
- reranker 批处理。
- 尽量用抽取式压缩。
- 延迟预算清晰，例如检索 100ms、rerank 300ms、生成 2s。

### 2. 高质量分析链路

适合研究、法务、复杂知识问答：

```text
query -> query decomposition -> multi-hop retrieval -> rerank per subquery -> evidence clustering -> LLM compression with citations -> answer with references
```

特点：

- 允许更高延迟。
- 关注覆盖率和引用完整性。
- 需要证据聚类和冲突检测。
- 适合异步任务或报告生成。

### 3. 长上下文模型链路

长上下文模型并不意味着取消 rerank：

```text
query -> broad recall -> coarse rerank -> document-level packing -> section-level compression -> long-context answer
```

原因：

- 长上下文成本更高。
- 噪声越多，模型越可能被无关内容干扰。
- 注意力并不等于可靠阅读，重要证据仍要靠前放置。

---

## 六、评估与调优

### 1. 检索排序侧指标

| 指标 | 含义 | 关注点 |
| --- | --- | --- |
| Recall@K | 正确证据是否进入前 K | 初召覆盖能力 |
| Precision@K | 前 K 中相关证据比例 | 噪声控制 |
| MRR | 第一个正确证据排名 | 首条证据质量 |
| NDCG | 考虑相关性等级和排序位置 | 排序整体质量 |
| Context Precision | 注入上下文中有多少是相关内容 | prompt 噪声 |
| Context Recall | 答案所需证据是否被注入 | 可回答性 |

### 2. 生成侧指标

| 指标 | 含义 |
| --- | --- |
| Faithfulness | 答案是否被上下文支持 |
| Answer Correctness | 答案是否正确 |
| Citation Accuracy | 引用是否真的支持对应结论 |
| Abstention Rate | 没有证据时是否能拒答 |
| Latency / Cost | 重排与压缩带来的额外开销 |

### 3. 典型调优顺序

如果 RAG 效果差，不要一上来换大模型。建议按顺序排查：

1. 标准答案需要的原文是否已经入库。
2. chunk 切分是否保留了完整语义。
3. 初召 top_100 是否能找到正确证据。
4. reranker 是否把正确证据排到前面。
5. 去重和 MMR 是否误删了关键证据。
6. 上下文压缩是否丢失了数字、条件、例外。
7. prompt 是否要求基于证据回答并输出引用。
8. 生成模型是否仍然幻觉或忽略证据。

---

## 七、高频面试问题与参考答案

### 1. Reranker 和 Embedding 检索有什么区别？

Embedding 检索通常是第一阶段召回，把 query 和 chunk 编码成向量后做近似最近邻搜索，目标是快和覆盖面广。Reranker 通常是第二阶段精排，对 query 和候选文本做更细粒度匹配，目标是把真正能回答问题的候选排到前面。

面试可以补一句：Embedding 更像“从百万文档中粗筛几十个候选”，Reranker 更像“从几十个候选中挑出最该进入 prompt 的几个证据”。

### 2. 为什么 Cross-Encoder Reranker 效果通常比向量相似度好？

因为 Cross-Encoder 会同时看 query 和 passage，让两者 token 之间直接交互，因此能识别限定条件、否定关系、实体对应关系和答案是否完整。向量检索把 query 和 passage 分别编码，速度快，但信息在向量里被压缩，细粒度判断能力弱。

缺点是 Cross-Encoder 成本更高，所以一般只对初召后的少量候选做重排。

### 3. Reranker 候选数应该怎么选？

没有固定值，要结合召回质量、延迟预算和文档规模。常见做法是初召 50-200 个候选，rerank 20-100 个候选，最终注入 3-10 个片段。

如果正确证据经常不在 rerank 输入里，说明初召 top_k 太小或召回策略有问题。如果正确证据能进入 rerank，但排不到前面，说明 reranker、分数融合或训练数据需要优化。

### 4. 多路召回后如何合并排序？

先对每一路结果保留来源分数和 rank，再按稳定 chunk_id、文本指纹、文档版本做去重。然后对分数做归一化，融合向量分、BM25 分、reranker 分、标题命中、新鲜度、权威性等信号。

需要强调：权限过滤必须在排序前完成，不能把无权限文档放进候选后再靠降权处理。

### 5. MMR 解决什么问题？

MMR 解决 top_k 结果高度重复的问题。它在选择候选时同时考虑与 query 的相关性，以及与已选候选的差异性。

适合多方面问题、对比问题和总结问题；不适合单一精确事实问答，因为这类问题通常更需要最权威、最直接的一条证据。

### 6. 上下文压缩和摘要有什么区别？

上下文压缩的目标是减少 token，同时保留回答所需证据。摘要只是其中一种手段。压缩还可以通过句子窗口、结构化抽取、去重合并、表格行筛选等方式完成。

面试中要强调：生成式摘要可能引入幻觉或丢失细节，关键业务场景要保留原文引用和 span，不能只把摘要作为唯一证据。

### 7. 长上下文模型出现后，还需要 reranker 吗？

需要。长上下文只是扩大容量，不等于提高证据质量。把大量噪声放进上下文会增加成本、延迟和错误引用风险。Reranker 仍然负责把最相关、最可信的证据放到更靠前的位置。

可以回答：长上下文降低了“放不下”的问题，但没有解决“该放什么、按什么顺序放、哪些内容是噪声”的问题。

### 8. 如何判断是召回问题、排序问题还是生成问题？

可以分层排查：

1. 看正确文档是否在索引中。如果不在，是数据摄取或索引问题。
2. 看初召 top_100 是否包含正确 chunk。如果没有，是召回问题。
3. 看 rerank 后正确 chunk 是否进入 top_5。如果没有，是排序问题。
4. 看最终 prompt 是否保留了关键证据。如果没有，是压缩或组装问题。
5. 看 prompt 有证据但答案仍错。如果是这样，才重点看生成模型和提示词。

### 9. Reranker 分数低时应该怎么办？

常见策略：

- 低于阈值时拒答，提示“当前知识库没有足够依据”。
- 扩大召回范围，例如增加 top_k、启用 BM25 或多查询改写。
- 降级到澄清问题，让用户补充条件。
- 对高风险场景转人工或返回候选引用让用户自行确认。

不要在低置信证据上强行生成答案，否则很容易幻觉。

### 10. 如何评估上下文压缩有没有伤害效果？

要对比压缩前后的端到端效果：

- 正确答案是否仍能从压缩上下文中推出。
- 数字、日期、版本、否定条件是否被保留。
- 引用是否仍能指向原文。
- token、延迟、成本是否下降。
- Faithfulness 和 Citation Accuracy 是否下降。

如果压缩后答案更短但引用错误增加，说明压缩策略损害了证据保真。

### 11. RAG 结果重复很多，应该怎么优化？

可以从三个层面处理：

1. 索引层：减少 chunk overlap 过大、去除重复文档、处理历史版本。
2. 召回层：按 chunk_id、文本 hash、文档版本去重。
3. 排序层：使用 MMR 或每个文档设置最大入选片段数。

如果重复来自同一文档相邻 chunk，可以考虑相邻合并，保留完整上下文，而不是简单删除。

### 12. LLM Rerank 适合生产吗？

适合部分场景，但不应默认作为主链路。LLM Rerank 灵活、解释性强，适合小候选集、复杂判断、离线标注辅助或高价值低频任务。但它成本高、延迟高、输出稳定性不如专用 reranker。

生产中更常见的是：专用 reranker 做主排序，LLM Rerank 用于难例分析、离线评测或少量复杂 query 的增强路径。

---

## 八、面试答题模板

### 问：你会如何设计一个高质量 RAG 检索排序链路？

可以按这个结构回答：

1. **召回层**：混合检索，向量召回负责语义，BM25 负责关键词和编号，metadata 做权限、租户、时间过滤。
2. **候选治理**：多路结果合并，保留来源分数，按 chunk_id、版本、文本指纹去重。
3. **重排序层**：用 Cross-Encoder reranker 对候选精排，并融合标题、新鲜度、权威性等业务信号。
4. **多样性控制**：对总结、对比类问题用 MMR，避免上下文被重复片段占满。
5. **上下文压缩**：按 token budget 做句子窗口、结构化抽取或摘要压缩，保留引用和原文 span。
6. **拒答机制**：低分或证据不足时拒答或澄清，不强行生成。
7. **评估闭环**：用 Recall@K、NDCG、Context Precision、Faithfulness、Citation Accuracy 和线上反馈持续调参。

### 问：如果线上用户反馈“答案引用了无关文档”，你怎么排查？

参考回答：

1. 先看最终 prompt，确认无关文档是否真的被注入。
2. 如果注入了，查看它来自哪一路召回，是向量、BM25、多查询还是规则推荐。
3. 看 reranker 分数和最终融合分，判断是 reranker 判断错，还是业务规则把它推高。
4. 检查 metadata 过滤，确认租户、权限、版本和文档类型是否正确。
5. 检查上下文压缩，确认是否把原文裁剪成了容易误解的片段。
6. 增加 badcase 标注，把该问题加入回归评测集。

### 问：如何在低延迟和高质量之间权衡？

参考回答：

- 限制 rerank 候选数，先用轻量规则过滤候选。
- 对 reranker 做 batch 推理或部署独立服务。
- 对常见 query 缓存召回和重排结果。
- 根据 query 类型走不同链路：简单事实问答走快速链路，复杂分析走高质量链路。
- 设置超时和降级策略，例如 reranker 超时后使用混合分数排序，但高风险场景不能无脑降级生成。

---

## 九、常见误区

1. **以为 top_k 越大越好**
   top_k 增大可能提高召回，但也会增加噪声、成本和延迟。

2. **把长上下文当成排序替代品**
   长上下文能装更多内容，但不能判断哪些内容应该被信任。

3. **把摘要当原文证据**
   摘要可以帮助压缩，但引用和审计应回到原文 span。

4. **忽略分数尺度差异**
   向量分、BM25 分、reranker 分不能直接相加，必须归一化或校准。

5. **用降权代替权限过滤**
   无权限内容不应该进入候选集，更不应该进入 prompt。

6. **只优化生成模型，不看检索链路**
   如果正确证据没进上下文，再强的模型也容易编。

---

## 十、复习清单

面试前确认自己能回答：

- Embedding 检索和 Reranker 的区别是什么？
- Cross-Encoder 为什么更准，为什么更慢？
- 多路召回后如何合并、去重和融合分数？
- MMR 解决什么问题，什么时候不适合用？
- 上下文压缩有哪些方式，各自风险是什么？
- 为什么长上下文模型仍然需要检索排序？
- 如何判断 RAG 问题发生在召回、排序、压缩还是生成？
- 如何设计低延迟、高质量、可回归评估的 RAG 排序链路？

如果只能记住三句话：

1. 初召追求覆盖，重排追求精度，压缩追求在 token 预算内保留证据。
2. RAG 质量的关键不是“塞更多上下文”，而是“塞更可信、更相关、更可溯源的上下文”。
3. 排序和压缩必须用评估闭环验证，否则很容易把正确证据删掉或把噪声推到前面。
