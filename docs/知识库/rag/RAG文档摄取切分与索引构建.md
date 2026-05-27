# AI 应用开发面试主题：RAG 文档摄取、切分与索引构建流水线

## 核心概念

**文档摄取（Ingestion）** 是把 PDF、Confluence、数据库 FAQ 等外部知识接入 RAG 的过程，不只是上传文件，还要留下可追溯、可更新、可鉴权的元数据。

**文档解析（Parsing）** 把原始格式转成结构化块（标题、段落、表格、页码），而不是一上来压成纯字符串——后续切分才能利用结构。

**清洗与规范化** 去掉页眉页脚、乱码、重复模板，但 **保留** 条款号、表头、错误码、产品型号等对检索关键的字段。

**Chunk** 是检索与注入的基本单元，通常含正文、`title_path`、`document_id`、页码、ACL、版本、`content_hash`。

**索引构建** 往往同时维护：原文库、chunk 库、向量索引、关键词索引、metadata 索引；生产环境用 **版本化发布**（alias 切换），避免边建边污染线上。

一句话：**RAG 线上效果很多输在离线**——解析错、切分碎、旧 chunk 未删、metadata 缺失，后面模型再强也救不回来。

```json
{
  "chunk_id": "policy-2026-001#003",
  "document_id": "policy-2026-001",
  "title_path": ["售后政策", "退款条件"],
  "text": "用户在订单支付后 7 天内可申请退款...",
  "page_start": 3,
  "tenant_id": "tenant_a",
  "acl": ["support"],
  "content_hash": "a6b1..."
}
```

---

## 核心知识点

### 1. 推荐流水线（面试常让画）

```text
连接器同步 -> 原文落库 -> 解析(structured blocks) -> 清洗
  -> 结构化切分 -> chunk 质检 -> 批量 Embedding
  -> 向量/关键词/metadata 索引 -> 回归评测 -> alias 发布
```

要点：原文可重放；索引带版本；失败可断点续跑（状态机：DISCOVERED → PARSED → CHUNKED → EMBEDDED → PUBLISHED）。

### 2. 切分策略对比

| 策略 | 场景 | 风险 |
| --- | --- | --- |
| 固定长度 | 原型、纯文本 | 切断语义（面试减分项） |
| 按标题/段落 | 制度、产品文档 | 超长节需二次拆 |
| 递归切分 | 通用 | 参数要评测 |
| 语义切分 | 长文主题多变 | 成本高 |
| 父子 chunk | 要精准召回+完整回答 | 存储与组装更复杂 |

**父子 chunk**：小块召回，命中后拉父章节进 Prompt；适合政策、手册，但父块不能无脑全塞。

### 3. chunk size / overlap

不报死「500 字」——先结构切，再用评测集调。FAQ 常一条一块；操作文档步骤与前置条件放一起；合同按条款+页码。指标：Recall@K、可回答率、重复率、平均注入 token、引用准确率。

### 4. Metadata（安全与溯源基础）

必备：`tenant_id`、`acl`、`document_id`、`chunk_id`、`source_uri`、`title_path`、`page/section`、`updated_at`、`version`、`content_hash`。  
**硬约束**（租户、权限）必须在检索层前置；软约束（时间排序）可 rerank。

### 5. 增量更新与去重

- 变更检测：etag、文件 hash、内容 hash。
- 更新：按 `document_id` 删旧 chunk 再写；或 chunk 级 diff 只重嵌变更部分。
- 删除：同步向量、BM25、metadata；软删除 + 异步 compact。
- 去重：文件 hash → 段落 hash → 语义相似 → 检索后按 document 合并 top_k。

### 6. 表格 / 图片 / 代码

表格保留表头与行列语义，可转自然语言描述 chunk；扫描件 OCR 记置信度，低置信勿进高风险问答；代码块 metadata 含 `language`、`file_path`、`symbol`，避免截断函数。

### 7. 索引发布与评估

```text
index_v1(线上) -> 构建 v2 -> 质检+Recall 回归 -> 小流量 -> 切 alias
```

离线指标：解析成功率、metadata 完整率、chunk 重复率；检索指标：Recall@K、引用页码准确率；线上：文档更新到可检索的延迟、过期知识坏 Case。

```python
def validate_chunk(chunk: dict) -> list[str]:
    errors = []
    text = chunk.get("text", "").strip()
    if len(text) < 20:
        errors.append("文本过短，不适合单独入库")
    if len(text) > 3000:
        errors.append("文本过长，向量语义易被稀释")
    for field in ["document_id", "chunk_id", "source_uri", "tenant_id"]:
        if not chunk.get(field):
            errors.append(f"缺少必要元数据：{field}")
    return errors
```

```python
def ingest_document(doc) -> None:
    raw_store.save(doc)  # 先落原文，解析策略变更可重放
    parsed = parser.parse(doc)
    chunks = chunker.split(cleaner.normalize(parsed.blocks), metadata=doc.metadata)
    valid = [c for c in chunks if not validate_chunk(c)]
    embeddings = embed_client.batch_embed([c["text"] for c in valid])
    index_writer.upsert(valid, embeddings, index_version=doc.metadata["version"])
```

---

## 高频面试问题与标准答案

### 1. 如何设计企业知识库的摄取与索引流程？

**标准答案：**  
我会做成可观测的流水线：连接器拉原文并落库，记下 document_id、tenant、acl、version。解析阶段按格式保留标题、表格、页码；清洗去噪声但保留条款号和表头。切分优先结构切，超长递归，需要时用父子 chunk。入库前做长度和 metadata 质检，再批量 Embedding，同时写向量和 BM25。发布用 index 版本，新版本跑 Recall 回归和小流量后再切 alias，能回滚。每个阶段记成功率、耗时和 parser/chunker 版本，方便定位坏 Case。

### 2. chunk size 怎么设置？

**标准答案：**  
我不给全行业统一的字数。先看文档类型和问题类型：FAQ 一条一块，政策按条款，代码按函数。先结构切，再对评测集调 size 和 overlap。召回片段经常缺背景说明切太碎；片段很长但答不上可能是块太大或主题混杂。我会看 Recall@K、可回答率和引用准确率一起调。

### 3. 为什么必须保留 metadata？

**标准答案：**  
metadata 是权限、过滤、溯源和增量的控制面。tenant 和 acl 决定谁能看到；document_id/chunk_id 支持更新删除；source_uri 和 page 支持用户核验；version 和 updated_at 避免答过期政策；content_hash 支持去重和增量 diff。没有这些字段，向量效果再好也难上企业生产。

### 4. 文档更新后如何避免召回旧内容？

**标准答案：**  
稳定 document_id + version；同步时 hash 检测变化；更新时删该文档全部旧 chunk 再写新版，或 chunk diff；检索默认只查当前生效版本；索引用 alias，新索引评测通过再切换；删除要清向量和关键词索引，不能只删对象存储里的 PDF。

### 5. PDF 解析差会怎样？怎么处理？

**标准答案：**  
会导致切分乱、表格丢结构、OCR 错字、页码缺失，表现是片段「看起来相关」但缺关键条件。要区分文字层 PDF 和扫描件；文字层用版面分析保留结构；扫描件 OCR 带置信度；页眉页脚规则清洗；表格转保留语义的文本；关键合同低置信走人工审核。

### 6. 父子 chunk 是什么？何时用？

**标准答案：**  
小 chunk 负责精准命中，父 chunk 或父章节提供完整上下文，比如命中「7 天退款」后带上整节「售后政策」。适合手册、政策、长文档。代价是索引和 Prompt 组装更复杂，父块要控 token，不能整章硬塞。

### 7. 摄取流水线如何做可观测性？

**标准答案：**  
分阶段指标：同步的新增/修改/删除和延迟；解析成功率、OCR 置信度；切分的数量、长短分布、重复率；Embedding 失败重试；发布的版本和评测结果。线上坏 Case 要能关联 chunk_id、parser_version、chunker_version，这样知道是内容变了还是流水线变了。

### 8. 如何避免重复内容刷屏 top_k？

**标准答案：**  
文件级 hash 防重复上传；段落级 hash 去模板；语义级 SimHash/Embedding 去轻微改写；检索后按 document_id 或相似度合并。历史版本可审计存储，但默认检索只查当前版，别把旧版和新版一起进 top_k。

---

## 面试回答加分点

- 强调 **「知识入库质量决定上限」**，不只谈向量模型。
- 解析输出 **structured blocks**，反对「一律固定 500 字切」。
- 说清楚 **权限 metadata 在检索前**，不靠 Prompt 挡泄露。
- 提到 **index 版本、alias、回滚** 和增量删旧 chunk。
- 表格、扫描件、代码有 **专门处理**，体现真实项目经验。
- 评估不只盯最终答案，还看 **解析成功率、metadata 完整率、Recall@K**。
