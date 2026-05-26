# LLM 应用成本、缓存与限流设计

## 核心概念

### 1. 成本拆解

LLM 线上成本不仅是生成费用，还包括：

| 类别 | 组成 | 优化方向 |
| --- | --- | --- |
| 输入 token | 系统 Prompt、历史、RAG 上下文 | 裁剪、压缩、分层检索 |
| 输出 token | 回答长度 | max_tokens、格式约束 |
| 辅助调用 | Embedding、rerank、审核、改写 | 缓存、批处理、小模型 |

原则：**先观测再优化**，不能只说“换便宜模型”。

### 2. 缓存类型

| 类型 | 适用 | 风险 |
| --- | --- | --- |
| 精确缓存 | FAQ、分类、固定抽取 | 版本/权限过期 |
| 语义缓存 | 相似问题复用 | 意图不同、越权命中 |
| 检索缓存 | RAG 召回/rerank 结果 | 知识库更新未失效 |
| 前缀缓存 | 固定系统 Prompt、长文档前缀 | 模型/Prompt 版本变化 |

缓存 key 必须包含：**权限范围、Prompt 版本、模型版本、任务类型**。

### 3. 限流与预算

按用户、租户、接口、模型等级治理 **RPM、TPM、并发数、日预算、重试次数**。限流目标是**保护系统容量**与**公平分配额度**，不是简单 403。

---

## 核心知识点

### 1. 成本优化步骤

```text
建立基线 → 设定预算 → 控制输入 → 控制输出 → 加缓存 → 分级路由 → 降级兜底
```

```python
def cache_key(model: str, prompt_ver: str, scope: str, query: str) -> str:
  # 中文注释：scope 含租户/权限，避免越权或旧 Prompt 污染
  return f"{model}:{prompt_ver}:{scope}:{normalize(query)}"
```

### 2. 输入 token 控制

- 历史对话：只保留相关轮次或摘要。
- RAG：检索 → rerank → 只注入必要片段，控制 top_k。
- 系统 Prompt：去重复规则，稳定部分可前缀缓存。

### 3. 输出 token 控制

```python
request = {
  "model": "fast-chat",
  "messages": messages,
  "max_tokens": 600,  # 中文注释：按任务设上限，避免无限续写
  "temperature": 0.2,
}
```

Prompt 中约束条数与格式（如“最多 5 条，每条 ≤50 字”）。

### 4. 语义缓存设计

```python
def semantic_lookup(query: str, scope: str, threshold: float = 0.92):
  embedding = embed(normalize(query))
  hit = vector_cache.search(embedding, filter={"scope": scope}, top_k=1)
  if hit and hit.score >= threshold:
    return hit.answer  # 中文注释：高风险场景可只缓存检索，不缓存最终答案
  return None
```

阈值过高命中率低，过低误命中高；**权限不同不得共享缓存**。

### 5. 限流实现

```python
from dataclasses import dataclass

@dataclass
class Quota:
  rpm: int
  tpm: int
  daily_budget_usd: float

def check_quota(tenant_id: str, est_tokens: int) -> bool:
  usage = meter.get_tenant_usage(tenant_id)
  # 中文注释：超预算时降级或排队，而非无限重试
  if usage.tokens_today + est_tokens > quota.tpm * 60:
    return False
  return usage.cost_today < quota.daily_budget_usd
```

令牌桶适合突发流量；滑动窗口适合精确 RPM。

### 6. 分级路由与降级

- 简单任务 → 小模型；复杂/低置信 → 升级强模型。
- 超预算：排队、异步、缩短上下文、关闭非核心链路。
- 重试有上限，区分 429（退避）与 4xx 业务错误（不重试）。

### 7. 监控指标

- 成本：按场景/租户/模型的 token 与费用分布，关注 **P95 长尾**。
- 缓存：命中率、误命中率、过期率。
- 限流：拒绝率、排队时长、降级次数。

---

## 高频面试问题与标准答案

**Q1：如何降低线上 LLM 成本？**

拆成本来源 → 减少无效上下文 → 限制输出 → 精确/语义缓存高频请求 → 简单任务小模型 → Embedding 批处理 → 用监控验证每层收益。

**Q2：语义缓存有什么风险？**

相似≠等价：意图不同、权限不同、数据过期、个性化丢失。解决：设阈值、缓存 key 含权限与版本；高风险只缓存检索结果不缓存最终答案。

**Q3：流量突增如何处理？**

队列削峰 + 租户级限流 + 模型降级 + 关闭非核心链路 + 热点缓存 + 熔断。区分保护系统（拒绝/排队）与保护体验（高价值用户保留额度）。

**Q4：缓存和实验/发布如何配合？**

缓存 key 含 prompt_version、model_version、rag_policy_version、experiment_id，避免 A/B 污染或新 Prompt 命中旧答案。

**Q5：重试会不会放大成本？**

会。只对可恢复错误有限重试，指数退避；流式已输出不重试；监控重试率与重试带来的 token 增量。

**Q6：如何给租户分配预算？**

按租户等级设 RPM/TPM/日预算；核心功能保留保底额度；低优先级任务异步化；超预算时降级而非静默失败。

---

## 面试回答加分点

1. 强调**成本 = 输入 + 输出 + 辅助链路**，不只谈模型单价。
2. 主动说**权限过滤发生在缓存命中之前**，体现安全意识。
3. 用 **P95 长尾** 说明平均成本误导，体现生产化思维。
4. 语义缓存讲清**阈值与 scope**，不只说“用向量相似度”。
5. 限流与**降级策略**绑定：排队、异步、短答、转人工。
6. 重试与**成本放大**、**流式重复**一并提及。
7. 建立**观测 → 优化 → 验证**闭环，改完看命中率和质量指标。
