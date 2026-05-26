# LLM 应用性能与端到端延迟优化

## 核心概念

### 1. 端到端延迟

从用户发起到响应完成的总耗时，通常包含：

```text
鉴权 → 会话加载 → 检索/rerank → 工具 → Prompt 组装 → 模型排队+推理 → 后处理 → 返回前端
```

专业表达：**分段拆解**，不单说“模型慢”。

### 2. TTFT 与完整耗时

| 指标 | 含义 | 优化手段 |
| --- | --- | --- |
| TTFT | 首 token 时间 | 减输入 token、流式、减排队 |
| Time To Last Token | 完整生成结束 | 控输出 token、decode 速度 |
| 感知延迟 | 用户主观等待 | 流式、进度提示 |

流式**降低感知**，不一定减少完整推理时间。

### 3. P50 / P95 / P99

平均延迟易被长尾掩盖；SLA 常用 **P95/P99** 描述线上体验。

### 4. 吞吐与 Token 预算

瓶颈常在 RPM/TPM、检索、rerank、工具、连接池。**第一原则：减少无效 token**（输入与输出）。

---

## 核心知识点

### 1. 延迟分段诊断

| 分段 | 常见瓶颈 |
| --- | --- |
| 上下文加载 | 历史过长、Redis/DB 慢 |
| 检索 | top_k 过大、rerank 慢 |
| 工具 | 串行调用、外部 API 慢 |
| 模型 | 排队、输入/输出 token 多 |
| 前端/网关 | SSE 缓冲、等完整响应才渲染 |

```python
import time
from contextlib import contextmanager

class Trace:
  def __init__(self):
    self.spans = []

  @contextmanager
  def span(self, name: str):
    t0 = time.perf_counter()
    yield
    # 中文注释：记录耗时并关联 request_id、token 数
    self.spans.append({"name": name, "ms": round((time.perf_counter() - t0) * 1000, 2)})
```

### 2. 延迟预算示例

| 阶段 | 预算 |
| --- | --- |
| 鉴权 | 50ms |
| 向量检索 | 150ms |
| rerank | 300ms |
| LLM TTFT | 800ms |
| 完整流式答案 | 3–8s |

超标时快速定位责任段。

### 3. 减少输入 token

历史裁剪/摘要；RAG 先检索再压缩；去冗余系统 Prompt；工具结果**结构化摘要**而非整段 JSON。

### 4. 控制输出 token

```python
request = {"model": "fast-chat", "messages": messages, "max_tokens": 600}
```

Prompt 约束条数；高风险任务（合同/医疗）不能为快漏风险。

### 5. 流式实现要点

- 尽快 flush 首 chunk；网关**不缓冲** SSE。
- 记录 TTFT、chunk 间隔、取消率。
- 下单/写库等须等工具真实结果，不能“边流边执行”。

```python
def stream_answer(llm_stream):
  for chunk in llm_stream:
    if chunk.text:
      yield f"data: {chunk.text}\n\n"
  yield "event: done\ndata: {}\n\n"
```

### 6. 并发与并行

适合并行：多路只读检索、独立只读工具。  
不适合：触发限流的多模型竞赛、有依赖的写工具链。

```python
import asyncio

async def build_context(query: str):
  tasks = [
    asyncio.create_task(vector_search(query)),
    asyncio.create_task(keyword_search(query)),
  ]
  return merge(await asyncio.gather(*tasks, return_exceptions=True))
```

### 7. 缓存（带权限）

```python
def build_cache_key(req) -> str:
  return ":".join([
    req.tenant_id, req.permission_hash,
    req.knowledge_base_version, req.prompt_version,
    req.task_type, normalize(req.query),
  ])
```

### 8. 超时、重试、降级

总 deadline → 各阶段 timeout → 超时取消下游 → 降级（短答/异步/备用模型）。读可重试，写需幂等；避免无限重试引发雪崩。

### 9. RAG 性能

Embedding 缓存；索引分区；混合检索并行；轻重 rerank 分层；控制 top_k 并用评测集验证召回。

### 10. 验证优化效果

对比优化前后 P50/P95/TTFT/成本 + **质量指标**（准确率、引用命中率）。单条样例“感觉快了”不算数。

---

## 高频面试问题与标准答案

**Q1：响应慢怎么排查？**

端到端 trace 分段；看 P95/P99；TTFT 高查模型排队与输入 token；TTFT 正常但总时长高查输出 token 与工具；模型不慢则查检索、网关缓冲、前端。

**Q2：TTFT 和完整耗时的区别？**

TTFT 影响“有没有反应”；完整耗时影响任务完成。流式主要优化前者。

**Q3：为什么看 P95 不看平均？**

少量长尾（长上下文、工具超时、排队）拖垮体验；P95/P99 反映 SLA。

**Q4：如何减少输入 token？**

会话摘要；RAG 只注入必要证据；压缩 Prompt；工具结果摘要。用评测集验证质量未降。

**Q5：流式能解决性能问题吗？**

主要解决感知延迟；结构化输出和高风险写操作须等真实结果；确保代理不缓冲。

**Q6：缓存 key 怎么设计？**

含 tenant、权限、知识库版本、Prompt/模型版本、task_type、规范化 query；权限数据慎做全局响应缓存。

**Q7：RAG 延迟怎么优化？**

拆 embedding/检索/rerank/压缩各段；并行混合检索；分层 rerank；top_k 用评测平衡召回与延迟。

**Q8：Agent 工具慢怎么办？**

看 trace 是否串行；只读并行；批量接口；工具结果摘要；写工具不盲目并行；设超时与降级。

**Q9：低延迟与高质量如何权衡？**

任务分级；快模型 + 低置信升级强模型；记录路由原因与质量指标。

**Q10：优化后如何证明没伤质量？**

固定评估集 + 灰度，同时看延迟与准确率/引用率/格式通过率。

---

## 面试回答加分点

1. 口诀：**先打点，后优化；先 token，后换模型**。
2. 系统设计题顺序：指标 → trace → 瓶颈 → 短期（流式/裁剪/缓存）→ 中期（路由/RAG 分层）→ 稳定性（超时/熔断）→ 质量回归。
3. 主动说流式是**体验优化**，写操作须**确定性完成**。
4. 并行强调**独立、可合并、失败可降级、下游有容量**。
5. 缓存与**权限/版本**绑定，体现安全与发布治理意识。
6. 压测要模拟**真实分布**（长上下文、多工具、结构化输出）。
7. 反例：无限重试、top_k 一味调小、只看后端忽略网关/前端。
