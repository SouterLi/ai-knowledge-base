# RAG 增量索引与数据同步一致性

## 主题选择记录

- **本次主题序号**：第 43 篇。
- **README 位置**：追加到知识库目录表第 42 篇之后，归入「RAG 工程化」卷。
- **选择原因**：已有 RAG 主题覆盖了系统总览、文档摄取切分、向量索引、重排序、查询理解、答案生成、权限过滤和 GraphRAG，但线上面试经常继续追问：**知识库每天都在变，如何增量更新？旧向量怎么删？索引构建失败怎么回滚？用户为什么还搜到过期答案？** 本篇专门展开 RAG 上线后的增量同步、索引版本和一致性治理。
- **避免重复**：本文不重复讲 RAG 整体架构、Embedding 模型选型、rerank 或引用生成；也不重复 ACL 检索中的权限表达。本文聚焦 **数据源变更事件、增量切片、向量 upsert/delete、索引版本、缓存失效、读写一致性和回滚**。

## 核心概念

### 1. 什么是 RAG 增量索引

RAG 增量索引是指知识库数据发生新增、修改、删除、权限变化时，只处理受影响的文档或切片，并把变化同步到检索系统，而不是每次全量重建全部索引。

一个典型链路是：

```text
数据源变更
  -> 采集变更事件
  -> 判断文档版本和内容哈希
  -> 重新解析受影响文档
  -> 生成切片和 embedding
  -> upsert 新切片
  -> tombstone 或删除旧切片
  -> 切换索引版本或刷新别名
  -> 失效相关缓存
  -> 记录同步水位和审计日志
```

面试里要强调：**增量索引不是简单追加向量，而是要保证新增可见、更新不脏读、删除能传播、失败可重试、版本可回滚。**

### 2. 数据同步的一致性边界

RAG 一致性通常不是强事务一致性，而是工程上可解释的最终一致性。关键是把一致性边界讲清楚：

| 一致性对象 | 需要保证什么 | 常见问题 |
| --- | --- | --- |
| 文档元数据 | 标题、来源、租户、权限、更新时间准确 | 权限变了但向量元数据没变 |
| 切片内容 | 新旧 chunk 不混用 | 更新后同时召回旧段落和新段落 |
| 向量索引 | upsert/delete 与文档版本匹配 | 删除文档仍被召回 |
| 关键词索引 | BM25 与向量索引版本一致 | 混合检索两边结果冲突 |
| 缓存 | 检索缓存、答案缓存按知识库版本失效 | 用户继续命中过期答案 |
| 引用与审计 | 回答能追溯到当时的文档版本 | 复盘时找不到原始证据 |

面试表达可以这样说：**我会把知识库版本、文档版本和切片版本都放进检索结果与缓存 key，而不是只存 doc_id。**

### 3. 新增、修改、删除的差异

增量同步里最容易被低估的是删除。新增通常只需要解析、切片、写入；修改需要替换旧版本；删除不仅要删向量，还要处理缓存、引用、权限和审计。

| 变更类型 | 处理重点 | 风险 |
| --- | --- | --- |
| 新增 | 生成稳定 doc_id、chunk_id、embedding | 重复消费导致重复切片 |
| 修改 | 内容哈希对比、旧版本失效、新版本可见 | 新旧版本同时被召回 |
| 删除 | tombstone、索引 delete、缓存失效、审计留痕 | 已删除内容继续回答 |
| 权限变化 | 更新 metadata filter 或重建受影响切片 | 用户越权检索 |
| 批量导入 | 水位、幂等、分批提交、失败恢复 | 一批失败污染线上索引 |

一句话总结：**新增看幂等，修改看版本，删除看传播，权限变化看过滤条件和缓存失效。**

### 4. Tombstone 与物理删除

很多向量库支持 delete，但生产上通常还会引入 tombstone，也就是先把旧切片标记为不可检索，再异步物理清理。

这样做有三个好处：

1. **避免构建失败时丢数据**：新版本写入成功前，旧版本可以暂时保留但不暴露。
2. **支持回滚和审计**：短期内能知道某个切片为什么不可见。
3. **兼容不同索引实现**：有些 ANN 索引物理删除成本高，先逻辑删除更稳。

但 tombstone 不是永远保留。系统需要定期 compaction，清理过期向量、旧 embedding、旧原文快照和缓存引用。

### 5. 索引版本与别名切换

当更新范围较小时，可以对线上索引做增量 upsert；当更新范围很大、切片策略变化、Embedding 模型变化或权限模型变化时，更适合构建新索引版本，再通过 alias 切换。

```text
kb_prod_alias -> kb_index_v12

后台构建 kb_index_v13
  -> 校验文档数、切片数、召回样例、权限过滤
  -> 小流量验证
  -> alias 原子切换到 kb_index_v13
  -> 保留 v12 用于回滚
```

面试重点是：**索引版本切换要像应用发布一样有灰度、校验和回滚，而不是把半成品索引直接暴露给线上请求。**

## 核心知识点

### 1. 文档、切片和版本标识要稳定

增量同步的基础是稳定 ID 和版本字段。常见设计：

| 字段 | 含义 |
| --- | --- |
| `doc_id` | 业务稳定文档 ID，不随内容变化 |
| `doc_version` | 文档内容版本或更新时间水位 |
| `content_hash` | 文档正文哈希，用于判断是否需要重切 |
| `chunk_id` | 稳定切片 ID，通常由 doc_id、切片序号、切片策略版本生成 |
| `chunk_version` | 切片版本，避免新旧切片混用 |
| `embedding_model` | 向量模型版本 |
| `index_version` | 知识库索引版本 |
| `acl_version` | 权限快照版本 |

示例：

```python
import hashlib


def build_chunk_id(doc_id: str, chunk_index: int, splitter_version: str) -> str:
    # 中文注释：chunk_id 要稳定，便于重复消费同一事件时覆盖而不是插入重复向量
    raw = f"{doc_id}:{chunk_index}:{splitter_version}"
    return hashlib.sha256(raw.encode("utf-8")).hexdigest()[:24]


def content_hash(text: str) -> str:
    # 中文注释：内容未变化时可以跳过重新切片和重新向量化
    normalized = "\n".join(line.strip() for line in text.splitlines() if line.strip())
    return hashlib.sha256(normalized.encode("utf-8")).hexdigest()
```

如果面试官问“为什么不用随机 UUID”，可以回答：随机 UUID 让重试和覆盖变难，重复消费时会产生重复向量；稳定 ID 配合 upsert 才能保证幂等。

### 2. 增量任务要按事件水位推进

企业数据源通常来自 CMS、飞书、Confluence、数据库、对象存储或消息队列。不能只靠定时全量扫目录，应该保存同步水位：

| 水位类型 | 适用场景 |
| --- | --- |
| `updated_at` | 数据库、CMS |
| `event_offset` | Kafka、消息队列 |
| `etag` / `version_id` | 对象存储 |
| `change_token` | 第三方文档系统 |
| `snapshot_id` | 批量导入或离线任务 |

简化的事件处理逻辑：

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class ChangeEvent:
    doc_id: str
    op: str
    version: int
    source_offset: int


def should_process(event: ChangeEvent, stored_version: int | None) -> bool:
    # 中文注释：旧事件和重复事件都跳过，保证乱序消费时不会回滚到旧内容
    if stored_version is None:
        return True
    return event.version > stored_version
```

高频追问是“事件乱序怎么办”。回答要点：按 `doc_id` 做分区、记录已处理版本、旧版本事件直接丢弃；如果第三方源不保证顺序，就在落库前做版本比较或拉取最新快照确认。

### 3. 更新文档时避免新旧切片混查

文档修改后，切片数量可能变化。旧版本 5 个 chunk，新版本 3 个 chunk，如果只 upsert 前 3 个，旧的第 4、5 个还会被召回。

更稳妥的做法：

1. 新版本切片写入临时状态 `building`。
2. 新版本全部 embedding 成功后，标记为 `active`。
3. 同一 `doc_id` 的旧版本切片标记为 `expired`。
4. 检索过滤只允许 `status = active` 且 `doc_version = latest_visible_version`。
5. 定时任务清理 expired 切片。

伪代码：

```python
def publish_new_doc_version(vector_store, doc_id: str, new_version: int, chunks: list[dict]) -> None:
    # 中文注释：先写新版本，再失效旧版本，避免中间失败导致文档完全不可检索
    for chunk in chunks:
        vector_store.upsert(
            id=chunk["chunk_id"],
            vector=chunk["embedding"],
            metadata={
                "doc_id": doc_id,
                "doc_version": new_version,
                "status": "building",
            },
            text=chunk["text"],
        )

    vector_store.update_metadata(
        filter={"doc_id": doc_id, "doc_version": new_version},
        values={"status": "active"},
    )
    vector_store.update_metadata(
        filter={"doc_id": doc_id, "doc_version": {"$lt": new_version}},
        values={"status": "expired"},
    )
```

如果向量库不支持批量 metadata update，就需要通过文档元数据表控制 latest visible version，并在检索时 join 或 filter。

### 4. 删除要覆盖检索、缓存和答案引用

删除传播是面试重点。删除文档时至少处理四件事：

1. **文档元数据表**：状态改为 deleted，记录删除时间、操作者、原因。
2. **检索索引**：向量索引和关键词索引都要 tombstone 或 delete。
3. **缓存系统**：失效包含该 doc_id、tenant_id、kb_version 的检索缓存和答案缓存。
4. **审计与引用**：历史回答可以保留引用快照用于审计，但新请求不能再召回。

示例：

```python
def handle_delete(doc_id: str, tenant_id: str, vector_store, cache) -> None:
    # 中文注释：删除不是只删数据库，还要阻断检索入口并清理相关缓存
    vector_store.update_metadata(
        filter={"doc_id": doc_id, "tenant_id": tenant_id},
        values={"status": "deleted"},
    )
    cache.invalidate_by_tags([
        f"doc:{doc_id}",
        f"tenant:{tenant_id}:retrieval",
        f"tenant:{tenant_id}:answer",
    ])
```

面试表达要补充：如果涉及合规删除或用户撤回授权，不能只靠缓存过期，要主动失效并验证索引中已经不可检索。

### 5. 混合检索要保证多索引同版本

很多 RAG 会同时用向量检索、BM25、结构化过滤、知识图谱或 rerank。如果向量索引已经更新，但 BM25 仍是旧版本，混合召回就可能冲突。

常见方案：

- 给每个索引写入统一的 `kb_version`。
- 查询时只合并同一 `kb_version` 的结果。
- 切换版本时使用 alias 或 manifest 文件描述当前可见索引集合。
- 如果某个索引落后，降级为只使用已验证索引，并在回答中降低置信度或提示延迟。

一个 manifest 可以长这样：

```json
{
  "tenant_id": "t-001",
  "kb_version": "2026-06-14-001",
  "vector_index": "kb_t001_vec_v18",
  "bm25_index": "kb_t001_bm25_v18",
  "graph_index": "kb_t001_graph_v18",
  "embedding_model": "bge-m3-2025-09",
  "splitter_version": "semantic-v3",
  "visible_at": "2026-06-14T01:00:00Z"
}
```

面试官问“为什么要 manifest”，可以回答：它把一次知识库发布变成可审计的版本快照，查询、回放和回滚都能找到同一组索引。

### 6. 缓存 key 必须包含知识库版本

RAG 常见缓存包括 query 改写缓存、embedding 缓存、检索结果缓存、rerank 结果缓存、最终答案缓存。只按用户问题做 key 会导致知识库更新后继续返回旧答案。

更合理的 key：

```text
retrieval:{tenant_id}:{user_acl_hash}:{kb_version}:{query_hash}:{retriever_config_hash}
answer:{tenant_id}:{user_acl_hash}:{kb_version}:{prompt_version}:{model_version}:{query_hash}
```

重点：

- `tenant_id` 防止租户串数据；
- `user_acl_hash` 防止权限变化后命中旧结果；
- `kb_version` 防止知识库更新后命中过期召回；
- `prompt_version` 和 `model_version` 防止模型或模板升级后答案不一致；
- `retriever_config_hash` 防止 top_k、rerank、过滤策略变化后仍命中旧召回。

面试中可以说：**缓存是性能优化，不是数据一致性的豁免。只要知识库、权限或 Prompt 版本变了，相关缓存就必须失效或隔离。**

### 7. 大批量更新要走影子索引和灰度校验

当发生以下变化时，不建议直接在线 upsert：

- Embedding 模型升级；
- 切片策略升级；
- 权限模型大改；
- 文档规模大批量迁移；
- 向量库参数或索引类型变化；
- 多语言、多模态字段重建。

更安全的流程：

1. 构建影子索引 `kb_index_v_next`。
2. 对比文档数、切片数、embedding 成功率、平均 chunk 长度。
3. 跑固定评测集，检查召回率、MRR、引用准确率和权限过滤。
4. 小流量 shadow query，对比线上索引召回差异。
5. 通过 alias 原子切换。
6. 保留上一版本用于快速回滚。

面试表达：**小改用增量 upsert，大改用新索引版本；判断标准是变化会不会影响大量切片表示或检索排序。**

### 8. 监控指标要能发现“数据不一致”

RAG 增量同步不只看任务成功失败，还要看数据质量和线上表现：

| 指标 | 说明 |
| --- | --- |
| change lag | 数据源变更到线上可检索的延迟 |
| event retry rate | 事件处理重试率 |
| embedding failure rate | 向量化失败率 |
| stale hit rate | 召回过期 chunk 的比例 |
| deleted hit count | 已删除文档被召回次数 |
| version mismatch count | 向量索引、BM25、manifest 版本不一致次数 |
| cache stale hit | 知识库更新后仍命中过期缓存次数 |
| zero-result rate | 更新后无结果率异常升高 |

如果面试官问“用户反馈搜到旧答案怎么办”，排查顺序可以是：

1. 查回答 trace 中的 `kb_version`、`doc_id`、`chunk_version`。
2. 查文档源的当前版本和删除状态。
3. 查向量索引 metadata 是否仍 active。
4. 查 BM25 或混合检索是否召回旧版本。
5. 查缓存 key 是否缺少 kb_version 或 acl_version。
6. 查前端或网关是否复用了旧会话上下文。

### 9. 失败恢复要靠幂等和补偿

增量同步链路长，任何步骤都可能失败。设计上要保证每一步可重试：

- 事件消费幂等：相同 `event_id` 或更旧 `doc_version` 不重复处理。
- 切片幂等：相同输入和切片策略得到相同 chunk_id。
- upsert 幂等：同一 chunk_id 覆盖写入。
- 发布幂等：同一 doc_version 多次发布结果一致。
- 删除幂等：重复 tombstone 不报错。
- 补偿任务：定期对比源文档表和索引 manifest，修复漏同步。

对账任务示例：

```sql
SELECT d.doc_id
FROM source_documents d
LEFT JOIN indexed_documents i
  ON d.doc_id = i.doc_id
WHERE d.status = 'active'
  AND d.doc_version > COALESCE(i.doc_version, 0);
```

这类查询用于找出“源文档已经更新，但索引还没追上”的文档，再投递补偿任务。

### 10. 权限变化也是索引变更

很多系统只在文档内容变化时重建索引，却忽略权限变化。实际上权限变化可能比内容变化更敏感。

常见做法：

- 权限字段放 metadata，检索时强制 filter；
- 权限版本 `acl_version` 进入缓存 key；
- 组织架构或用户组变化后，刷新 ACL 快照；
- 高敏场景不把大规模用户列表写进每个 chunk，而是用 doc_id 召回后再二次鉴权；
- 删除授权时主动失效缓存，并对高敏知识库做抽样验证。

面试重点：**权限过滤必须由系统侧执行，不能让模型根据 Prompt 自己判断用户能不能看。**

## 高频面试问题与标准答案

### Q1：RAG 知识库更新后，为什么用户还会搜到旧答案？你怎么排查？

标准答案：

我会先从一次回答的 trace 入手，看它命中的 `kb_version`、`doc_id`、`chunk_id` 和缓存信息。旧答案通常有几类原因：第一，源文档更新了，但增量同步任务还没处理到，change lag 太高；第二，新切片写进去了，但旧切片没有 expired 或 delete，导致新旧版本混查；第三，向量索引更新了，但 BM25 或 rerank 缓存还是旧版本；第四，答案缓存的 key 没包含知识库版本或权限版本。排查时我会先验证当前检索是否还能召回旧 chunk，再看缓存命中和索引 metadata，最后检查同步水位和失败重试日志。

### Q2：你会怎么设计 RAG 的增量索引流程？

标准答案：

我会把流程拆成变更采集、版本判断、解析切片、向量化、索引发布和缓存失效几步。数据源侧通过 `updated_at`、消息队列 offset 或对象存储 etag 记录变更水位；处理时先比较 doc_version 和 content_hash，没变化就跳过；有变化就生成稳定 chunk_id，写入新版本切片。新版本全部成功后再把旧版本标记为 expired，并刷新 manifest 或索引别名。最后按 doc_id、tenant、kb_version 失效检索缓存和答案缓存。整个过程要保证幂等，重复消费同一事件不能产生重复向量。

### Q3：文档删除时，只从数据库删掉记录够不够？

标准答案：

不够。RAG 删除要覆盖源文档、向量索引、关键词索引、缓存和审计。数据库删了，但向量库里旧 chunk 还在，用户仍然可能召回；答案缓存没失效，也可能继续返回旧内容。生产上我通常先把文档状态改成 deleted，再对相关 chunk 做 tombstone 或 delete，同时失效带有这个 doc_id 或 tenant 的缓存。历史回答引用可以按合规要求保留审计快照，但新请求必须不能再召回这份内容。

### Q4：为什么修改文档时会出现新旧内容同时被召回？

标准答案：

一般是因为只 upsert 了新切片，没有处理旧切片。比如旧版本有 8 个 chunk，新版本变成 5 个 chunk，前 5 个覆盖了，但旧的第 6 到第 8 个还在 active 状态。解决办法是引入 doc_version 和 chunk_version，检索时只允许 latest visible version；发布新版本时先写入新 chunk，全部成功后激活新版本，再把同一 doc_id 的旧版本统一标记为 expired。这样不会因为切片数量变化产生脏召回。

### Q5：什么时候用增量 upsert，什么时候全量重建索引？

标准答案：

小范围的文档新增、修改、删除适合增量 upsert，因为成本低、可快速可见。全量重建或影子索引适合影响全局表示的变化，比如 Embedding 模型升级、切片策略调整、权限模型大改、向量索引参数变化，或者大批量迁移。我的判断标准是：这次变化是否会影响大量 chunk 的向量表示、排序或过滤逻辑。如果会，就构建新索引版本，跑评测和 shadow query 后通过 alias 切换，并保留旧版本回滚。

### Q6：如何保证增量索引任务可以安全重试？

标准答案：

核心是幂等。事件要有 event_id、doc_id 和 doc_version，旧版本事件直接跳过；chunk_id 要由 doc_id、chunk_index 和 splitter_version 生成，不能用随机 ID；向量写入用 upsert，重复执行是覆盖而不是追加；删除用 tombstone，重复删除不报错；发布时记录 latest visible version，只有新版本完整成功才切换。除此之外还需要补偿任务，定期对账源文档版本和索引版本，修复漏处理的事件。

### Q7：如何处理消息乱序导致旧版本覆盖新版本？

标准答案：

我不会只按事件到达顺序写索引，而是按文档版本做条件更新。每个事件带 doc_version 或 source update time，处理前先查当前已发布版本。如果事件版本小于等于当前版本，就直接丢弃；如果版本更新，才进入切片和发布。消息队列可以按 doc_id 分区减少乱序，但不能完全依赖队列顺序，最终落库和发布时还要做版本比较。

### Q8：RAG 检索缓存怎么设计，才能避免知识库更新后返回旧结果？

标准答案：

缓存 key 不能只包含 query。至少要包含 tenant_id、用户权限哈希、kb_version、retriever 配置版本；如果缓存最终答案，还要包含 prompt_version 和 model_version。知识库更新后可以通过提升 kb_version 让旧缓存自然隔离，也可以按 doc_id 或 tenant tag 主动失效。高敏数据或权限频繁变化的场景，我会缩短缓存 TTL，甚至只缓存 embedding 或中间结果，不缓存最终答案。

### Q9：向量索引和 BM25 索引版本不一致怎么办？

标准答案：

最好用 manifest 管理一次知识库发布对应的索引集合，比如 vector index、BM25 index、graph index 都对应同一个 kb_version。查询时只合并同版本结果。如果发现某个索引构建落后，可以临时降级为只使用已验证的索引，或者继续读上一稳定版本，而不是把两个版本混在一起。版本不一致会让结果很难解释，尤其是引用和 rerank 结果会冲突。

### Q10：如何验证一次增量索引发布是成功的？

标准答案：

我会分数据完整性、检索质量和权限安全三类验证。数据上看文档数、chunk 数、embedding 成功率、失败重试和 change lag；质量上跑固定 query 集，看 recall、MRR、引用命中和 no-result rate 是否异常；权限上抽样验证不同租户和用户组不能越权召回。大版本发布还要做 shadow query，对比新旧索引召回差异。只有这些指标通过，才把 manifest 或 alias 切到新版本。

### Q11：如果用户要求“删除我的数据”，RAG 系统要注意什么？

标准答案：

这属于合规删除，不能只删业务表。需要找出这份数据进入过哪些地方：原文存储、解析后的文本、向量索引、关键词索引、缓存、评测集、日志和人工审核记录。对线上检索要立即 tombstone 并失效缓存；对审计日志要按合规策略脱敏或保留最小必要信息；删除后要做验证查询，确认对应 doc_id 不再可检索。面试里我会特别强调，删除传播要有任务状态和审计记录，否则无法证明已经处理完成。

### Q12：RAG 增量同步和普通后端数据同步最大的区别是什么？

标准答案：

普通后端同步通常关注结构化记录是否一致，RAG 还要关注“派生数据”是否一致。一个文档会派生成切片、embedding、向量索引、BM25 索引、rerank 缓存、答案缓存和引用证据。只更新源表不代表检索结果已经更新。所以 RAG 同步要管理 doc_version、chunk_version、embedding_model、index_version 和缓存版本，并且要验证召回质量和权限过滤。这也是为什么 RAG 知识库发布更像一次数据产品发布，而不是简单的数据写入。
