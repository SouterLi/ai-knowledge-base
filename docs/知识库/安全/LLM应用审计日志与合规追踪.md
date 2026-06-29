## 主题选择记录

- **本次序号**：第 51 篇。
- **README 位置**：追加到目录表末尾，归入「安全与合规」卷。
- **选题理由**：README 已有安全防护、身份权限、多租户隔离、隐私脱敏和红队评测，但面试中经常继续追问：**线上出事后怎么追溯？怎么证明某次回答用了哪些 Prompt、知识片段、工具和权限？日志既要排障又不能泄露隐私，怎么设计？合规审计需要留哪些证据？** 本篇单独展开 LLM 应用审计日志与合规追踪。
- **避免重复**：不重复讲通用 Prompt 注入、防越权、脱敏算法和红队样本设计；本文聚焦 **审计事件模型、Trace 链路、日志脱敏、留存策略、合规取证、面试表达**。

## 核心概念

### 1. 什么是 LLM 应用审计日志

LLM 应用审计日志不是普通的 debug log，而是为了**事后追溯、合规证明、风险定位和责任界定**记录的关键事实。

普通日志关注“系统有没有报错”；审计日志关注：

- 谁发起了请求；
- 使用了什么身份、租户和权限；
- 命中了哪个 Prompt、模型、RAG 策略和工具；
- 访问了哪些知识片段或业务资源；
- 模型生成了什么类型的结果；
- 系统为什么允许、拒绝、脱敏、审批或回滚。

一句面试表达：**LLM 审计的目标不是把所有上下文原样存下来，而是用最小必要证据还原一次 AI 决策链路。**

### 2. Trace、日志和审计的区别

| 概念 | 主要目的 | 典型内容 | 面试重点 |
| --- | --- | --- | --- |
| Trace | 串起一次请求的调用链 | span、耗时、上下游调用、错误 | 定位链路问题 |
| 运行日志 | 辅助开发和运维排障 | 异常栈、参数摘要、状态码 | 控制噪声和敏感信息 |
| 审计日志 | 合规、追责、风控取证 | 身份、资源、策略、版本、决策 | 不可抵赖、可查询、可留存 |

在 LLM 应用里，三者需要关联到同一个 `trace_id` 或 `request_id`。否则线上出了问题，只知道“模型答错了”，却查不到当时检索了哪些 chunk、工具参数是什么、权限策略是否命中。

### 3. 为什么 LLM 审计更复杂

传统 Web 系统常见审计对象是接口、数据库和用户操作；LLM 应用还多了几类不稳定因素：

- Prompt 会变，且 Prompt 版本会影响模型行为；
- 模型会变，同一请求在不同模型别名下输出不同；
- RAG 召回会变，知识库、切分、索引、重排策略都会影响上下文；
- Agent 会调用工具，工具参数可能由模型生成；
- 输出是自然语言，是否违规、是否幻觉需要语义判断；
- 日志里可能包含用户隐私、企业知识、系统 Prompt 和工具返回的敏感字段。

所以面试中要强调：**LLM 审计不是简单存一行 access log，而是围绕“模型输入、检索依据、工具动作、输出处理、策略版本”建立证据链。**

### 4. 审计设计的核心矛盾

LLM 审计的核心矛盾是 **可追溯性、隐私安全、成本和可用性**：

- 记录太少：出事后无法复盘，也无法向客户或合规说明。
- 记录太多：Prompt、PII、密钥、商业数据进入日志，形成二次泄露。
- 保留太久：合规和存储成本上升，数据删除请求难处理。
- 查询太慢：安全事件发生时无法快速定位影响范围。

成熟设计通常采用：**结构化事件 + 敏感字段摘要/脱敏 + 分级留存 + 权限化查询 + Trace 关联**。

## 核心知识点

### 1. 推荐审计链路

```text
请求入口 → 鉴权与身份上下文 → Prompt/RAG/工具编排
       → 模型调用 → 输出校验与脱敏 → 响应返回
       → 结构化审计事件 → 安全存储 → 查询/告警/合规报表
```

面试中要说清楚：审计不是最后统一打个日志就够了，而是在关键决策点记录事实：

| 阶段 | 必记信息 | 不能只记什么 |
| --- | --- | --- |
| 请求入口 | user_id、tenant_id、roles、ip、request_id | 只记用户问题原文 |
| Prompt 组装 | prompt_version、变量摘要、策略版本 | 只记最终 Prompt 全文 |
| RAG 检索 | query_version、doc_id、chunk_id、score、acl_result | 只记“检索成功” |
| 工具调用 | tool_name、参数摘要、审批状态、执行结果码 | 只记模型说要调用工具 |
| 模型调用 | model_alias、provider、temperature、token、耗时 | 只记供应商返回成功 |
| 输出处理 | 安全检测结果、脱敏动作、拒答原因 | 只记最终答案 |

### 2. 审计事件要结构化

审计日志要能查询、聚合和回放，所以优先使用结构化事件，而不是自由文本。

```json
{
  "event_type": "llm_tool_call",
  "request_id": "req_20260629_001",
  "trace_id": "trace_8f3a",
  "tenant_id": "tenant_a",
  "user_id_hash": "u_9b12f",
  "session_id_hash": "s_41c2e",
  "actor_roles": ["sales_ops"],
  "model_alias": "chat_quality",
  "prompt_version": "sales_agent_prompt_v18",
  "tool_name": "crm_update_customer",
  "tool_args_digest": "sha256:7a9c...",
  "policy_result": "allowed_after_approval",
  "approval_id": "apv_10086",
  "resource_ids": ["customer_239"],
  "result_code": "success",
  "created_at": "2026-06-29T01:10:00Z"
}
```

这里的重点是：**敏感值存摘要或脱敏值，关键版本和决策结果必须可查**。

### 3. 关键字段设计

LLM 审计字段可以分为六类：

| 字段类别 | 示例 | 作用 |
| --- | --- | --- |
| 身份字段 | tenant_id、user_id_hash、roles、scopes | 确认谁在什么权限下操作 |
| 链路字段 | request_id、trace_id、span_id、session_id_hash | 串起一次请求和多轮对话 |
| 版本字段 | prompt_version、model_alias、rag_policy_version、guardrail_version | 复现当时行为 |
| 资源字段 | doc_id、chunk_id、tool_name、resource_id | 知道访问了什么 |
| 决策字段 | allow/deny、approval_id、mask_rule、reject_reason | 解释系统为什么这么处理 |
| 指标字段 | latency_ms、token_count、cost、risk_score | 做排障、告警和治理 |

面试金句：**没有版本字段的审计日志只能说明发生过请求，不能说明为什么当时会这样回答。**

### 4. Trace 需要覆盖 LLM 专有 Span

常见 Web Trace 只覆盖 HTTP 和数据库调用；LLM 应用需要扩展专有 span：

```text
request
  ├─ auth.resolve_identity
  ├─ config.resolve_prompt
  ├─ rag.rewrite_query
  ├─ rag.retrieve
  ├─ rag.rerank
  ├─ llm.generate
  ├─ agent.tool_call
  ├─ guardrail.output_check
  └─ response.send
```

每个 span 至少记录：输入摘要、输出摘要、版本、耗时、状态、错误码。不要把完整 Prompt 和工具返回直接放进 Trace 属性里，因为很多 APM 平台默认开放给较多工程人员查看。

### 5. Prompt 和上下文不要无脑全量落库

面试里常见追问是：“为了排障，是否应该把完整 Prompt 都存下来？”

推荐回答：**默认不全量存，按风险和场景分层处理**。

| 内容 | 建议做法 | 原因 |
| --- | --- | --- |
| 用户输入 | 脱敏后存摘要，必要时采样存原文 | 可能含 PII 和商业信息 |
| 系统 Prompt | 存版本号和 hash，不直接暴露全文 | 防止泄露安全策略 |
| RAG chunk | 存 doc_id/chunk_id/版本/score | 可通过受控权限回查原文 |
| 工具参数 | 存参数 schema、资源 id、敏感字段 hash | 兼顾追踪与隐私 |
| 模型输出 | 存输出摘要、风险标签、脱敏后文本 | 降低二次泄露 |

代码示例：

```python
import hashlib
from dataclasses import dataclass

@dataclass
class AuditInput:
  tenant_id: str
  user_id: str
  request_id: str
  prompt_version: str
  user_query: str
  retrieved_chunk_ids: list[str]

def digest(text: str) -> str:
  return hashlib.sha256(text.encode("utf-8")).hexdigest()

def mask_query(query: str) -> str:
  # 中文注释：真实系统应接入 PII 识别，这里只演示审计前先做最小化处理
  return query.replace("身份证号", "[PII_TYPE]")

def build_audit_event(data: AuditInput) -> dict:
  return {
    "event_type": "llm_request_context",
    "tenant_id": data.tenant_id,
    "user_id_hash": digest(data.user_id),
    "request_id": data.request_id,
    "prompt_version": data.prompt_version,
    "query_digest": digest(data.user_query),
    "query_masked": mask_query(data.user_query),
    "retrieved_chunk_ids": data.retrieved_chunk_ids,
  }
```

面试中可以补一句：如果金融、医疗等场景确实需要保留原文，也要做**加密存储、最小授权、访问审计、到期删除和客户数据删除响应**。

### 6. RAG 审计要能解释引用依据

RAG 场景最容易被问：“用户投诉答案错了，你怎么查？”

需要记录：

- 原始问题摘要和 query rewrite 版本；
- 向量检索、关键词检索、混合召回的策略版本；
- 召回的 `doc_id`、`chunk_id`、文档版本、分数；
- ACL 过滤结果：哪些文档被过滤，原因是什么；
- 重排模型和分数；
- 最终进入 Prompt 的 chunk 列表；
- 回答中的引用映射。

示例事件：

```json
{
  "event_type": "rag_context_selected",
  "request_id": "req_001",
  "query_rewrite_version": "rewrite_v6",
  "retrieval_policy": "hybrid_v4",
  "selected_chunks": [
    {
      "doc_id": "contract_88",
      "chunk_id": "contract_88_013",
      "doc_version": "2026-06-20",
      "score": 0.82,
      "acl": "allowed"
    }
  ],
  "filtered_chunks_count": 3,
  "filter_reason": "scope_denied"
}
```

面试表达：**RAG 审计不一定要保存知识原文，但必须保存可回查的证据索引，否则无法判断是召回错、权限错、重排错还是生成错。**

### 7. Agent 审计要记录工具决策链

Agent 场景不能只审计最终回复，因为真正的风险在工具执行。

```text
用户意图 → 模型选择工具 → 生成参数 → 服务端校验 → 审批/确认 → 工具执行 → 结果回注
```

审计事件需要覆盖：

- 模型建议调用的工具和理由摘要；
- 工具参数摘要、资源 id、参数校验结果；
- 权限校验结果和策略版本；
- 人工审批、用户确认或双人复核记录；
- 幂等键、重试次数和补偿结果；
- 工具返回中被脱敏的字段。

```python
def audit_tool_decision(trace_id: str, identity, tool_name: str, args: dict, decision) -> dict:
  # 中文注释：审计记录保留资源标识和参数摘要，不直接保存高敏参数原文
  return {
    "event_type": "agent_tool_decision",
    "trace_id": trace_id,
    "tenant_id": identity.tenant_id,
    "user_id_hash": digest(identity.user_id),
    "tool_name": tool_name,
    "resource_id": args.get("customer_id") or args.get("order_id"),
    "args_digest": digest(str(sorted(args.items()))),
    "policy_version": decision.policy_version,
    "decision": decision.status,
    "reason_code": decision.reason_code,
    "approval_required": decision.approval_required,
  }
```

面试重点：**模型的 Thought 不一定要全量落库，但工具选择、参数、权限、审批和执行结果必须可审计。**

### 8. 日志脱敏和访问控制同样重要

很多团队只关注用户侧数据安全，却忽略“日志系统也可能泄露”。审计平台要做：

- 写入前脱敏：PII、密钥、Token、银行卡、手机号、邮箱等；
- 字段分级：普通字段、敏感字段、高敏字段；
- 查询鉴权：不同角色能看的字段不同；
- 明文回查审批：高敏原文需要临时授权和审计；
- 日志导出限制：导出水印、审批、过期链接；
- 内部人员访问留痕：谁查了哪个用户、哪个租户、哪次请求。

面试表达：**审计日志本身也是敏感数据资产，不能因为是内部系统就默认全员可见。**

### 9. 留存周期和删除策略

审计日志要分级留存：

| 数据类型 | 保留建议 | 处理方式 |
| --- | --- | --- |
| 聚合指标 | 较长周期 | 不含个人敏感信息 |
| 结构化审计事件 | 按合规要求保留 | 加密、分区、可查询 |
| 脱敏 Prompt/输出 | 中等周期 | 采样、压缩、访问控制 |
| 原文内容 | 最短必要周期 | 强审批、到期删除 |
| 高敏工具结果 | 原则上不落库 | 仅存摘要和资源引用 |

如果用户提出删除请求，需要能根据 `tenant_id`、`user_id_hash`、`session_id_hash` 和资源索引定位相关记录，并区分“必须删除的数据”和“法规要求保留的审计证据”。

### 10. 合规追踪要支持正向和反向查询

正向查询：给定 `request_id`，查完整链路。

```text
request_id → 身份 → Prompt 版本 → RAG chunk → 工具调用 → 输出检测 → 响应
```

反向查询：给定用户、资源、Prompt 版本、模型版本或风险类型，查影响范围。

```text
prompt_version=v18 → 影响了哪些请求、租户、用户、投诉和坏 Case
doc_id=policy_2026 → 被哪些回答引用过
tool_name=refund_create → 哪些调用需要审批但被绕过
risk_type=pii_leak → 哪些输出触发过高危告警
```

面试中这点很加分：**事故复盘不只需要查单次请求，还要能反向评估影响范围。**

### 11. 审计和评测、告警、回放要打通

审计日志不是归档后没人看的数据。它应当进入治理闭环：

```text
线上审计事件 → 风险告警 → 人工复核 → 坏 Case 标注
          → 评测集回流 → Prompt/策略修复 → 回放验证
```

典型告警规则：

- 同一用户短时间多次触发敏感信息探测；
- 某 Prompt 版本上线后拒答率或敏感告警突增；
- 某工具出现异常高频调用或重复参数；
- RAG ACL 过滤失败或跨租户资源进入上下文；
- 高风险输出被脱敏后仍被用户复制导出。

### 12. 常见设计误区

| 误区 | 风险 | 正确做法 |
| --- | --- | --- |
| 全量保存 Prompt 和输出 | 日志二次泄露 | 默认摘要/脱敏，原文最小化 |
| 只记录最终答案 | 无法定位检索和工具问题 | 记录 RAG、工具、策略版本 |
| 没有 request_id | 多系统日志串不起来 | 全链路传递 trace_id/request_id |
| 没有版本号 | 无法复现当时行为 | 记录 Prompt、模型、RAG、Guardrail 版本 |
| 日志权限过宽 | 内部越权查看用户数据 | 字段级权限和访问审计 |
| 审计只写不查 | 事故时不可用 | 设计正向追踪和反向影响面查询 |

## 高频面试问题与标准答案

**Q1：LLM 应用为什么需要专门的审计日志？普通业务日志不够吗？**

标准答案：普通业务日志通常只能看到接口成功失败和异常栈，但 LLM 应用的问题经常发生在 Prompt、RAG、工具调用和输出策略里。比如一次错误回答，可能是 Prompt 版本变了、检索 chunk 错了、ACL 过滤漏了，也可能是工具参数被模型生成错了。所以我会设计专门的审计事件，记录身份、权限、Prompt 版本、模型别名、RAG chunk、工具调用、输出检测和策略决策。这样出问题后能还原决策链路，而不是只说“模型答错了”。

**Q2：审计日志里要不要保存完整 Prompt？**

标准答案：默认不建议无脑保存完整 Prompt。Prompt 里可能有系统策略、用户隐私、企业知识和工具返回，直接落日志会带来二次泄露。我一般会保存 Prompt 版本号、模板 hash、变量摘要、RAG chunk id 和必要的脱敏片段。确实需要保留原文的高合规场景，要做加密存储、严格授权、访问留痕和到期删除。面试里我会强调，审计的目标是可追溯，不是把所有敏感上下文都复制一份。

**Q3：如果用户投诉 RAG 答案不准确，你怎么通过审计定位问题？**

标准答案：我会先用 request_id 找到这次请求的完整 Trace，看原始问题、query rewrite 版本、召回策略、召回的 doc_id/chunk_id、重排分数和最终进入 Prompt 的上下文。然后检查这些 chunk 是否是最新文档、是否被 ACL 正确放行、回答引用是否来自这些 chunk。如果召回没有命中正确文档，是检索或索引问题；如果命中了但模型没用，是生成和 Prompt 问题；如果引用了无权文档，是权限过滤问题。这样可以把“答案错了”拆成可修复的链路问题。

**Q4：Agent 工具调用应该审计哪些内容？**

标准答案：Agent 场景重点不是最终回复，而是工具决策链。我会记录模型选择了什么工具、参数摘要、资源 id、权限校验结果、策略版本、是否需要审批、审批人或确认记录、幂等键、执行结果和补偿结果。敏感参数不直接明文落库，只保存摘要或脱敏值。这样既能复盘为什么调用了某个工具，也能证明服务端确实做了权限校验和审批，而不是完全相信模型。

**Q5：日志脱敏会不会影响排障？如何平衡？**

标准答案：会有影响，所以不能简单一刀切。我会把字段分级：低敏字段直接记录，高敏字段存 hash 或 token，原文通过受控资源 id 回查；只有在事故排查、合规取证等场景下，经过审批临时查看原文，并记录谁在什么时候查看了什么。这样日常排障依赖结构化字段、版本号、资源 id 和摘要；少数需要原文的情况再走强授权。核心是既能定位问题，又不让日志系统变成新的数据泄露源。

**Q6：怎么设计一次 LLM 请求的 request_id 和 trace_id？**

标准答案：入口网关生成 request_id 和 trace_id，并在 Prompt 组装、RAG 检索、模型调用、工具执行、输出检测和响应返回之间透传。request_id 更偏业务请求，trace_id 串起底层 span。如果是多轮对话，还要有 session_id，但日志里最好存 session_id_hash。这样单次请求可以正向追踪，多轮问题也能聚合分析，同时避免直接暴露用户会话标识。

**Q7：审计日志如何支持合规取证？**

标准答案：合规取证需要证明“谁在什么权限下访问了什么资源，系统依据什么策略做了什么决策”。所以审计日志要结构化、不可随意篡改、带时间戳和版本号，并能按 request_id、用户、租户、资源、Prompt 版本、工具名反查。敏感字段要加密和分级授权，访问审计本身也要留痕。这样面对客户质疑、监管检查或安全事件时，可以拿出完整证据链，而不是靠口头解释。

**Q8：如果某个 Prompt 版本上线后出现安全事故，你怎么评估影响范围？**

标准答案：我会用 Prompt 版本号反向查询审计日志，找出这个版本影响的租户、用户、请求数量、风险告警、工具调用和投诉记录。然后按风险类型筛选，比如是否涉及 PII 泄露、越权检索、违规输出或高危工具调用。必要时回放这些请求，用修复后的 Prompt 和策略重新跑评测。这里的关键是审计日志必须记录版本字段，否则只能查单个 case，无法评估整体影响面。

**Q9：你会把模型的中间推理过程也记录下来吗？**

标准答案：一般不会把完整推理过程作为默认审计内容。原因是它可能包含敏感信息，而且不同模型也不一定提供可靠的内部推理。工程上我更关注可验证的外部决策：模型输入摘要、Prompt 版本、RAG 证据、工具选择、参数、权限校验、输出检测和最终结果。如果是 ReAct 类 Agent，可以记录 action、observation 摘要和决策标签，但不必把所有自然语言思考原文长期保存。

**Q10：审计日志本身如何防篡改？**

标准答案：可以从三层做。第一，写入路径上只允许服务端统一写审计事件，业务方不能随意改历史日志。第二，存储上使用追加写、对象存储版本化、WORM 或哈希链等机制，关键事件可以计算签名。第三，访问和导出都要有权限控制和访问审计。面试时不用一上来堆技术名词，重点是说明审计日志要满足完整性、可追溯和最小权限。

**Q11：如何处理日志留存和用户删除请求之间的冲突？**

标准答案：我会先按数据类型分级：聚合指标和结构化审计事件可以保留较长周期，原文 Prompt、用户输入、工具返回这类敏感内容尽量短周期或不落库。用户删除请求到来时，通过用户 hash、租户和资源索引定位相关数据，删除或匿名化可删除部分；如果法规要求保留某些审计证据，也要把保留依据和访问限制记录清楚。核心是从设计上减少原文留存，降低后续删除和合规压力。

**Q12：面试中如果让你设计一套 LLM 审计系统，你会怎么答？**

标准答案：我会先说目标是可追溯、可合规、可排障，同时避免日志二次泄露。架构上在网关生成 request_id/trace_id，全链路透传；在身份解析、Prompt 组装、RAG 检索、工具调用、模型调用、输出检测这些关键点写结构化审计事件；字段上记录身份、资源、版本、策略决策和指标，敏感内容默认摘要或脱敏；存储上做加密、分级留存和权限查询；治理上支持正向追踪单次请求，也支持按 Prompt 版本、资源和风险类型反向查影响范围。这样回答能覆盖工程落地和合规风险。
