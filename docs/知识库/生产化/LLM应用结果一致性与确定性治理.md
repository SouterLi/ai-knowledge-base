# LLM 应用结果一致性与确定性治理

## 主题选择记录

- **本次序号**：第 33 篇。
- **README 位置**：追加在目录表第 32 篇之后，归入「生产化与 LLMOps」卷。
- **选题理由**：已有主题覆盖测试、发布、模型网关、评估、缓存、流式和坏 Case 闭环，但线上面试经常继续追问「为什么同一个问题两次回答不一样」「如何复现线上结果」「重试会不会造成业务不一致」。本篇聚焦 **结果一致性、可复现性、幂等与确定性边界**，作为生产化治理的小主题展开。
- **去重判断**：本主题不重复「LLM 应用测试与 Mock 策略」。测试篇重点是如何写自动化测试；本篇重点是线上系统如何让输出、工具副作用、版本、缓存和审计在工程上可解释、可复现、可治理。

## 核心概念

### 1. 结果一致性

结果一致性不是要求模型每次逐字输出完全相同，而是要求在业务可接受范围内保持稳定：

| 层级 | 一致性目标 | 示例 |
| --- | --- | --- |
| 格式一致 | 输出满足固定 Schema | JSON 字段齐全、类型正确 |
| 事实一致 | 关键事实不冲突 | 金额、日期、引用来源一致 |
| 决策一致 | 分类、审批、路由稳定 | 同一退款材料不应一会儿通过一会儿拒绝 |
| 副作用一致 | 重试不重复执行外部动作 | 退款、发券、发邮件只执行一次 |

面试里要强调：**自然语言措辞可以有差异，但业务事实、结构化字段、工具调用和外部副作用必须稳定**。

### 2. 确定性边界

LLM 本身是概率模型，即使 `temperature=0`，不同模型版本、推理后端、上下文截断、工具结果和系统 Prompt 都可能导致结果变化。因此工程上讨论确定性时，要区分：

- **可强约束部分**：输入组装、Prompt 版本、模型别名解析、工具参数校验、Schema 校验、业务规则、幂等键。
- **弱确定部分**：自然语言表达、开放式总结、创造性生成。
- **必须兜底部分**：高风险决策、金融金额、权限边界、合规拒答、外部写操作。

### 3. 可复现性

可复现性是线上排障的基础。一次请求要能回答：

```text
谁在什么版本下，用了什么输入、上下文、模型、参数、检索结果、工具结果，产生了什么输出？
```

如果日志只记录用户问题和最终答案，而没有 `prompt_version`、`model_version`、`config_version`、`retrieval_snapshot`，坏 Case 很难复盘。

### 4. 一致性治理的核心目标

一致性治理的目标不是把 LLM 变成传统规则引擎，而是把不确定性限制在可接受范围内：

```text
版本固定 → 输入快照 → 生成约束 → 输出校验 → 幂等执行 → 观测追踪 → 回归复现
```

## 核心知识点

### 1. 固定运行时版本

线上请求必须记录并尽量固定以下版本：

| 字段 | 作用 |
| --- | --- |
| `prompt_version` | 定位 Prompt 改动 |
| `model_alias` / `resolved_model` | 区分别名和真实模型 |
| `generation_params` | temperature、top_p、max_tokens、seed |
| `rag_policy_version` | top_k、阈值、重排策略 |
| `tool_schema_version` | 工具参数变化 |
| `config_version` | 运行时配置快照 |

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class RuntimeSnapshot:
    prompt_version: str
    model_alias: str
    resolved_model: str
    temperature: float
    top_p: float
    rag_policy_version: str
    tool_schema_version: str
    config_version: str

def build_trace_metadata(snapshot: RuntimeSnapshot) -> dict:
    # 中文注释：排障时依赖这份快照复现当时的模型行为
    return {
        "prompt_version": snapshot.prompt_version,
        "model_alias": snapshot.model_alias,
        "resolved_model": snapshot.resolved_model,
        "generation_params": {
            "temperature": snapshot.temperature,
            "top_p": snapshot.top_p,
        },
        "rag_policy_version": snapshot.rag_policy_version,
        "tool_schema_version": snapshot.tool_schema_version,
        "config_version": snapshot.config_version,
    }
```

面试表达重点：**只说 temperature=0 不够，版本和上下文也要固定**。

### 2. 输入快照与上下文哈希

同一个用户问题，如果历史消息、RAG 召回、工具结果或系统提示不同，输出不同是合理的。因此要记录输入快照：

- 用户输入原文与脱敏版本。
- 系统 Prompt 渲染后的文本或哈希。
- 历史消息截断后的实际内容。
- RAG 文档 ID、chunk ID、版本、分数和排序。
- 工具请求参数与响应摘要。

```python
import hashlib
import json

def stable_hash(payload: dict) -> str:
    # 中文注释：排序后再哈希，避免字典顺序导致快照 ID 不稳定
    raw = json.dumps(payload, ensure_ascii=False, sort_keys=True)
    return hashlib.sha256(raw.encode("utf-8")).hexdigest()

context_snapshot = {
    "system_prompt_hash": stable_hash({"text": rendered_system_prompt}),
    "history_ids": ["m101", "m102"],
    "retrieved_chunks": [
        {"doc_id": "policy_v3", "chunk_id": "c12", "score": 0.87},
    ],
    "tool_results": [
        {"tool": "get_order", "request_id": "tool_001", "status": "success"},
    ],
}
context_hash = stable_hash(context_snapshot)
```

### 3. 结构化输出优先于纯自然语言

需要稳定决策时，优先让模型输出结构化字段，再由服务端生成面向用户的文案。

```json
{
  "decision": "need_human_review",
  "confidence": 0.68,
  "risk_reasons": ["金额超过自动处理上限", "证据不足"],
  "user_message": "你的申请需要人工复核，请补充物流凭证。"
}
```

服务端应校验：

- `decision` 是否属于枚举值。
- `confidence` 是否在合法范围。
- 高风险 `decision` 是否触发人工审核。
- `risk_reasons` 是否有对应证据。

不要把「是否退款」「是否封禁」「是否放行」只藏在一段自然语言里。

### 4. 解码参数与随机性控制

常见参数策略：

| 场景 | 推荐参数 | 原因 |
| --- | --- | --- |
| 分类、抽取、路由 | 低 temperature、低 top_p | 降低随机性 |
| 创意文案 | 适度提高 temperature | 保留多样性 |
| 高风险决策 | 低随机 + 规则校验 + 人工兜底 | 模型不能单独拍板 |
| 评测回归 | 固定模型、Prompt、检索、参数 | 便于比较 |

注意：`seed` 如果供应商支持，也只是辅助复现，不能代替版本快照、输入快照和回归评测。

### 5. 重试策略不能破坏一致性

LLM 调用超时、429、5xx 可以重试，但要区分：

- **纯生成节点**：可重试，但需要记录多次候选和最终采用版本。
- **工具读操作**：可重试，需关注外部数据是否变化。
- **工具写操作**：必须先有幂等键，不能简单重跑。

```python
def execute_refund(order_id: str, amount: int, request_id: str) -> dict:
    existed = refund_store.get_by_request_id(request_id)
    if existed:
        return existed  # 中文注释：同一幂等键只返回历史结果，不重复退款

    result = payment_api.refund(order_id=order_id, amount=amount, idempotency_key=request_id)
    refund_store.save(request_id=request_id, order_id=order_id, result=result)
    return result
```

面试里可以说：**模型节点失败可以重试，外部副作用节点必须幂等或补偿**。

### 6. 缓存与一致性

缓存能提升稳定性和成本效率，但缓存 key 必须带上影响输出的变量：

```python
def llm_cache_key(
    tenant_id: str,
    user_scope: str,
    prompt_version: str,
    model_version: str,
    context_hash: str,
    query_hash: str,
) -> str:
    # 中文注释：权限、版本、上下文不同都不能共用缓存
    return ":".join([
        tenant_id,
        user_scope,
        prompt_version,
        model_version,
        context_hash,
        query_hash,
    ])
```

常见坑：

1. 只按用户问题缓存，导致权限越权或旧答案污染。
2. Prompt 升级后仍命中旧缓存。
3. RAG 文档更新后未失效。
4. A/B 实验中不同 variant 共享缓存，污染实验结果。

### 7. 输出校验与修复

输出校验分三层：

| 层级 | 校验内容 | 处理方式 |
| --- | --- | --- |
| Schema 校验 | JSON 是否可解析、字段类型 | 自动修复或重试一次 |
| 业务规则校验 | 金额、权限、状态机迁移 | 拒绝、降级或人工审核 |
| 事实一致性校验 | 答案是否被证据支持 | 重新检索、拒答、标记坏 Case |

高风险场景不要无限重试。重试次数过多会增加成本，也可能产生更多不一致结果。

### 8. 观测指标

一致性治理可以量化：

- `format_valid_rate`：结构化输出合法率。
- `decision_flip_rate`：同一输入快照下决策翻转率。
- `replay_match_rate`：回放请求与线上结果的关键字段匹配率。
- `idempotency_conflict_count`：幂等冲突次数。
- `cache_version_miss_rate`：因版本变化导致的缓存失效率。
- `manual_review_trigger_rate`：低置信或规则冲突进入人工比例。

面试中如果能提出「决策翻转率」和「回放匹配率」，通常比只说“加日志”更有工程深度。

### 9. 典型治理架构

```text
请求入口
  → 配置解析与版本快照
  → 输入组装与上下文哈希
  → 缓存查询
  → LLM / RAG / 工具编排
  → 结构化输出校验
  → 幂等执行或人工审核
  → Trace、指标、回放样本入库
```

各模块边界要清楚：

- **配置解析**负责确定版本。
- **上下文构建**负责确定实际输入。
- **LLM 调用层**负责参数和供应商适配。
- **执行层**负责权限、幂等和副作用。
- **观测层**负责复现和质量归因。

## 高频面试问题与标准答案

**Q1：同一个问题为什么 LLM 两次回答不一样？线上怎么治理？**

我会先区分原因：模型采样有随机性，temperature、top_p、模型版本、Prompt 版本、上下文截断、RAG 召回和工具返回都可能变化。线上治理不是只把 temperature 调成 0，而是固定运行时版本，记录输入快照和上下文哈希，对关键任务使用结构化输出和 Schema 校验。自然语言措辞可以略有差异，但事实字段、决策结果、工具调用和外部副作用要稳定。高风险场景还要加规则校验或人工审核。

**Q2：temperature=0 能保证确定性吗？**

不能完全保证。temperature=0 只能降低采样随机性，但模型服务端版本、推理实现、上下文内容、检索结果、工具结果变化都会影响输出。所以我一般把它作为一个手段，而不是确定性保证。真正可复现要同时记录 prompt_version、resolved_model、参数、输入快照、RAG chunk 版本和工具响应，必要时用回放测试验证关键字段是否一致。

**Q3：如何设计一套可复现的 LLM 请求日志？**

我会记录 request_id、tenant/user 脱敏标识、prompt_version、model_alias 和真实模型版本、生成参数、渲染后的 Prompt 哈希、历史截断结果、RAG 召回的文档和 chunk 版本、工具请求响应摘要、最终输出、校验结果、token、延迟和成本。重点是能回答「当时到底给模型看了什么，以及为什么走到这个结果」。隐私上要做脱敏和保留周期控制，不能为了复现把敏感数据无限期明文保存。

**Q4：结构化输出如何提升一致性？**

结构化输出把模型的不确定表达收敛成业务可校验的字段，比如 `decision`、`confidence`、`reason_codes`、`citations`。服务端可以检查枚举值、字段类型、置信度阈值和证据是否存在，再决定通过、拒绝、重试或进人工。这样用户文案可以灵活，但真正驱动业务流程的是可验证的结构化结果，不会靠解析一段自然语言来决定退款或审批。

**Q5：LLM 调用失败后重试要注意什么？**

要先判断节点有没有副作用。纯生成节点可以有限重试，并记录最终采用哪个结果；只读工具可以重试，但要关注外部数据是否变化；写操作不能简单重跑，必须带幂等键，比如同一个退款 request_id 多次执行只返回第一次结果。如果已经产生部分副作用，还要有状态机、补偿或人工介入。面试里我会明确说：重试解决临时失败，幂等解决重复执行，二者不是一回事。

**Q6：缓存会不会影响结果一致性？**

会，缓存既能提升稳定性，也可能制造错误一致性。缓存 key 必须包含租户、权限范围、Prompt 版本、模型版本、上下文哈希、实验 variant 和知识库版本。否则会出现 Prompt 已升级但还命中旧答案，或者用户 A 命中用户 B 权限下的结果。RAG 场景还要在文档更新后让相关缓存失效，不能只按用户 query 做简单缓存。

**Q7：如何衡量 LLM 应用的一致性治理效果？**

我会看几类指标：结构化输出合法率、同一输入快照下的决策翻转率、线上坏 Case 回放的关键字段匹配率、幂等冲突次数、缓存因版本变化的失效率，以及低置信进入人工审核的比例。单看成功率不够，因为 HTTP 200 不代表业务结果正确。更好的做法是把高频和高风险样本做成回放集，版本变更前后比较关键字段和业务决策。

**Q8：如果业务方要求“同一问题必须永远同一答案”，你怎么回应？**

我会先澄清“同一”的定义。如果是 FAQ 或固定政策类问题，可以通过固定知识版本、低随机参数、缓存和引用约束做到高度稳定；如果涉及实时订单、权限、库存或法规更新，答案随上下文变化是合理的。工程上可以承诺的是同一输入快照、同一版本、同一权限范围下，关键事实和决策应保持一致；而不是脱离时间、数据和版本承诺逐字相同。

**Q9：线上发现同一输入决策翻转，你会怎么排查？**

我会先拉 request_id 对比两次的运行时快照：Prompt、模型、参数、配置、实验分组是否一致；再看上下文哈希、RAG 召回、工具结果有没有变化。如果输入快照一致但决策不同，就检查采样参数、模型服务版本和输出校验策略；如果召回不同，就定位索引版本、重排分数或文档更新；如果工具结果不同，要看外部系统状态变化。最后把这个 case 加入回放集，避免后续版本再次翻转。

**Q10：一致性治理和测试有什么区别？**

测试更偏发布前验证，比如 Mock、黄金样例、回归评测；一致性治理覆盖线上运行时，包括版本快照、输入快照、缓存隔离、重试幂等、输出校验和观测回放。两者是配合关系：线上治理让坏 Case 可复现，复现后的样本进入测试集；测试通过后，线上仍要靠版本和 Trace 保证出了问题能定位和回滚。
