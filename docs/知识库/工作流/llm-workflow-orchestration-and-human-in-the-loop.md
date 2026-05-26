# AI 应用开发面试主题：LLM 工作流编排与 Human-in-the-loop

## 主题选择记录

- 本次主题：LLM 工作流编排与 Human-in-the-loop（人工审核/人工接管）
- 所属分类：AI 应用开发 / Workflow
- 已回顾仓库主题：RAG、Prompt 工程、流式与异步架构、模型网关、成本缓存限流、安全、Agent 规划执行、Agent 工具调用、上下文记忆、多模态、LLMOps。
- 去重说明：已有运行时文档重点讲流式响应和异步任务；已有 Agent 文档重点讲工具调用和规划可靠性。本主题重点讲 **多步骤业务流程如何编排、暂停、恢复、人工审核、补偿和审计**，不重复展开单次模型调用或工具 Schema 设计。
- 适用岗位：AI 应用开发工程师、LLM 应用工程师、AI 平台工程师、AI 工作流/Agent 平台工程师。

---

## 一、核心概念

LLM 应用一旦进入真实业务，通常不是「用户问一句，模型答一句」，而是一个可控流程：

```text
输入校验 → 意图识别 → 检索/工具调用 → 模型生成 → 规则校验 → 人工审核 → 结果发布 → 审计归档
```

**工作流编排**解决的问题是：把多个模型调用、规则节点、工具节点和人工节点组织成可观测、可恢复、可治理的业务流程。

面试中要明确区分三类系统：

| 类型 | 核心目标 | 典型场景 |
| --- | --- | --- |
| 普通 API 调用 | 请求进来后同步返回结果 | 简单问答、摘要、翻译 |
| 异步任务 | 长任务后台执行，前端查询进度 | 报告生成、批量分析 |
| 工作流编排 | 多节点、有分支、有状态、可暂停恢复 | 合同审查、客服工单、审批助手、营销内容生产 |

**一句面试表达：** LLM 工作流不是把多个 Prompt 串起来，而是把 AI 能力放进可审计、可回滚、可人工接管的业务状态机里。

---

## 二、核心知识点

### 1. 工作流的基本组成

一个可上线的 LLM 工作流通常包含：

1. **Trigger**：触发器，例如 API 请求、定时任务、消息队列、Webhook。
2. **Step/Node**：流程节点，例如模型调用、检索、规则判断、工具执行、人工审核。
3. **State**：流程状态，例如 `pending/running/waiting_for_review/succeeded/failed/cancelled`。
4. **Transition**：节点之间的跳转条件，例如分类结果、置信度、审核结论。
5. **Context**：跨节点传递的数据，例如用户输入、检索证据、模型草稿、审核意见。
6. **Policy**：权限、预算、重试、超时、风控、审计规则。

面试回答不要只说「用 LangChain/LangGraph/Dify」。更重要的是解释：**节点怎么定义，状态怎么存，失败怎么恢复，人工怎么介入，数据怎么审计**。

### 2. DAG、状态机与 Agent 的区别

| 维度 | DAG 工作流 | 状态机 | Agent 自主规划 |
| --- | --- | --- | --- |
| 流程形态 | 节点和边相对固定 | 状态和事件驱动迁移 | 模型动态决定下一步 |
| 可控性 | 高 | 很高 | 中等，依赖约束 |
| 适合场景 | ETL、内容生产、固定审核链路 | 审批、工单、交易流程 | 开放任务、复杂搜索、工具探索 |
| 风险点 | 分支过多导致维护困难 | 状态爆炸 | 幻觉、循环调用、越权工具 |
| 面试重点 | 依赖关系、并发、重试 | 状态一致性、幂等、补偿 | 工具边界、规划可靠性 |

实践中三者可以组合：外层用状态机管业务生命周期，中间用 DAG 编排确定性步骤，少数节点内部再用 Agent 处理开放式子任务。

### 3. 节点设计：输入、输出和错误协议

每个节点都应该像一个小型 API，有明确契约：

- 输入 Schema：节点需要哪些字段。
- 输出 Schema：节点产出哪些结构化结果。
- 错误类型：可重试错误、不可重试错误、业务拦截、人工介入。
- 超时与预算：最多运行多久、最多消耗多少 token。
- 审计字段：模型版本、Prompt 版本、工具版本、执行耗时、token 用量。

示例节点定义：

```python
from dataclasses import dataclass
from typing import Literal


@dataclass
class WorkflowStepResult:
    status: Literal["ok", "retryable_error", "blocked", "need_human"]
    output: dict
    error_code: str | None = None
    reason: str | None = None


def classify_ticket(ticket: dict) -> WorkflowStepResult:
    # 中文注释：真实系统中这里会调用模型，并把输出做 Schema 校验。
    category = "refund" if "退款" in ticket["content"] else "general"

    if ticket.get("vip") and category == "refund":
        return WorkflowStepResult(
            status="need_human",
            output={"category": category},
            reason="VIP 退款需要人工确认",
        )

    return WorkflowStepResult(status="ok", output={"category": category})
```

面试易错点：只保存最终答案，不保存节点级输入输出。这样线上出问题时无法判断是分类错、检索错、生成错，还是审核策略错。

### 4. Human-in-the-loop 的触发条件

人工介入不是「模型不行就找人」，而是风险治理机制。常见触发条件：

- **低置信度**：分类、抽取、审核结果低于阈值。
- **高风险操作**：退款、发邮件、改权限、发公告、执行代码。
- **合规敏感**：医疗、法律、金融、隐私、未成年人相关内容。
- **规则冲突**：模型结论和规则引擎结果不一致。
- **用户升级**：用户明确要求人工处理，或多轮未解决。
- **异常成本/延迟**：流程进入循环、工具连续失败、token 超预算。

人工节点至少要记录：

| 字段 | 作用 |
| --- | --- |
| `workflow_id` | 串联完整流程 |
| `step_id` | 定位等待人工的节点 |
| `review_reason` | 为什么需要人工 |
| `ai_suggestion` | 模型建议，供审核员参考 |
| `evidence` | 检索证据、工具返回、规则命中 |
| `reviewer_id` | 谁处理了 |
| `decision` | 通过、拒绝、修改、退回 |
| `decision_note` | 审核理由 |

### 5. 暂停、恢复与状态持久化

Human-in-the-loop 的关键是流程可以暂停。不能把状态只放在内存里，否则服务重启、Worker 崩溃或审核跨天都会丢失上下文。

推荐状态表设计：

```sql
CREATE TABLE workflow_runs (
  id TEXT PRIMARY KEY,
  workflow_name TEXT NOT NULL,
  status TEXT NOT NULL,
  current_step TEXT NOT NULL,
  context_json JSON NOT NULL,
  idempotency_key TEXT,
  created_by TEXT NOT NULL,
  created_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP NOT NULL
);

CREATE TABLE workflow_step_runs (
  id TEXT PRIMARY KEY,
  workflow_run_id TEXT NOT NULL,
  step_name TEXT NOT NULL,
  status TEXT NOT NULL,
  input_json JSON NOT NULL,
  output_json JSON,
  error_code TEXT,
  started_at TIMESTAMP,
  finished_at TIMESTAMP
);
```

恢复流程时，系统根据 `workflow_runs.current_step` 和 `context_json` 继续执行，而不是从头再跑一遍。

### 6. 重试、补偿与幂等

LLM 工作流里最容易被追问的是失败处理：

- **重试**：适合网络超时、模型 429、临时工具不可用。
- **不重试**：适合参数非法、权限不足、合规拦截。
- **补偿**：如果某个节点已经产生副作用，后续失败时要撤销或追加修正动作。
- **幂等**：同一个请求重试不能创建多个工单、重复扣费、重复发消息。

示例：

```python
def execute_refund_step(order_id: str, amount: int, request_id: str) -> dict:
    # 中文注释：先用幂等键检查是否执行过，避免重试造成重复退款。
    existing = find_refund_by_request_id(request_id)
    if existing:
        return existing

    refund = create_refund(order_id=order_id, amount=amount, request_id=request_id)
    return refund
```

面试中可以用这句话收束：**模型节点失败可以重试，业务副作用节点必须先讲幂等，再讲补偿**。

### 7. 工作流上下文管理

工作流上下文不是越大越好。要区分：

- **业务上下文**：订单号、用户 ID、工单状态、审核结论。必须结构化保存。
- **模型上下文**：Prompt、检索片段、模型草稿。用于追溯和复现。
- **临时上下文**：节点中间变量、缓存结果。可过期清理。
- **审计上下文**：谁在何时基于什么证据做了什么决定。必须可查询。

不要把所有历史消息原样塞给每个节点。更好的做法是每个节点只读取自己需要的结构化字段，并在调用模型前组装最小必要上下文。

### 8. 权限与审计

工作流平台通常会把 AI 能力接到真实业务系统，所以权限设计是面试高频点：

- 用户是否有权触发这个 workflow？
- 当前节点能不能调用某个工具？
- 人工审核员能不能看到这条数据？
- 模型建议是否允许直接执行高风险动作？
- 审计日志能否还原完整决策链？

关键原则：**模型不能绕过业务权限，人工审核也不能绕过数据权限**。

---

## 三、典型架构

```text
Client/API
   |
   v
Workflow Gateway  --鉴权/限流/幂等-->  Workflow DB
   |
   v
Queue / Scheduler
   |
   v
Worker / Orchestrator
   |-- LLM Provider
   |-- Retrieval Service
   |-- Business Tools
   |-- Rule Engine
   |
   v
Human Review Console
   |
   v
Audit Log / Metrics / Trace
```

各组件职责：

- **Workflow Gateway**：创建流程、校验权限、生成幂等键、返回 `workflow_id`。
- **Workflow DB**：保存流程状态、节点输入输出、审核结论。
- **Queue/Scheduler**：解耦请求和执行，支持延迟重试、定时恢复。
- **Worker/Orchestrator**：按节点定义推进流程，处理重试、分支和暂停。
- **Human Review Console**：展示 AI 建议、证据、风险原因，让人做决策。
- **Audit/Trace**：记录每个节点耗时、成本、输入输出摘要和错误。

---

## 四、面试高频问题与参考答案

### 1. LLM 工作流和普通异步任务有什么区别？

普通异步任务解决「耗时长」的问题，通常是创建任务、后台执行、查询结果。LLM 工作流还要解决「多步骤决策」的问题，包括分支、状态迁移、人工审核、节点级重试、补偿和审计。面试中可以说：异步任务是运行方式，工作流是业务过程建模，两者经常一起使用。

### 2. 什么情况下需要 Human-in-the-loop？

当模型输出会影响真实业务权益、合规风险或用户体验时就需要人工介入。典型条件是低置信度、高风险操作、规则冲突、敏感领域、用户升级和连续失败。人工介入要有明确触发规则，不能只是兜底口号。

### 3. 人工审核节点怎么设计？

核心是暂停和恢复。流程进入 `waiting_for_review` 后持久化当前上下文，审核台展示模型建议、证据和触发原因。审核员提交 `approve/reject/edit/return` 后，系统记录审核人和理由，再根据决策继续流转。关键是所有决策可追溯，且审核员权限也要校验。

### 4. 工作流执行到一半失败怎么办？

先区分错误类型。临时错误可以指数退避重试；参数、权限、合规错误直接失败或进入人工；产生副作用的节点要用幂等键避免重复执行，必要时设计补偿动作。恢复时从最近成功节点或当前等待节点继续，而不是无脑从头跑。

### 5. 为什么不能只用 Prompt 串联多个步骤？

Prompt 串联缺少工程边界：状态不清楚、失败难恢复、权限难控制、审计难还原、成本难统计。上线系统需要把步骤变成显式节点，每个节点有输入输出、错误协议、超时预算和日志。这样才能定位问题并做回归测试。

### 6. DAG 和状态机怎么选？

如果流程主要是固定依赖关系，例如「抽取 → 分类 → 生成 → 审核」，DAG 更直观。如果流程有明确业务生命周期，例如工单从 `open` 到 `pending_review` 再到 `resolved`，状态机更适合。复杂系统常用状态机管理外层生命周期，用 DAG 执行某个状态内的任务。

### 7. 如何避免 LLM 工作流进入死循环？

设置最大步数、最大重试次数、最大 token 预算和最大运行时间；对 Agent 子流程限制工具集合；对同类错误做熔断；连续失败后进入人工审核或失败终态。还要记录 trace，定位循环发生在哪个节点和哪类输入上。

### 8. 工作流上下文应该保存什么？

必须保存业务状态、节点输入输出、模型版本、Prompt 版本、检索证据、工具调用摘要、人工审核结论和错误信息。敏感字段要脱敏或按权限加密。不要只保存最终答案，否则无法复盘和合规审计。

### 9. 高风险工具能不能让模型自动调用？

可以让模型提出调用建议，但执行前必须经过应用层权限校验、参数校验、风控规则和必要的人工确认。面试回答要强调：模型只产生意图，不直接拥有业务系统权限；越高风险的动作，越需要审批、幂等和审计。

### 10. 如何测试 LLM 工作流？

分层测试：

- 节点单测：输入输出 Schema、错误分类、幂等逻辑。
- 流程测试：正常路径、分支路径、失败重试、人工审核恢复。
- 回归评测：固定样本下模型输出是否满足关键事实和格式要求。
- 混沌测试：模拟模型超时、工具 500、队列重复投递、Worker 重启。
- 权限测试：不同角色能否触发、审核和查看对应数据。

---

## 五、实践建议

- 先把业务流程画成状态机，再决定哪些节点使用 LLM。
- LLM 节点输出必须结构化，并经过 Schema 和业务规则校验。
- 人工审核要有触发条件、证据展示、决策记录和恢复机制。
- 对外部副作用工具先设计幂等键，再设计重试策略。
- 每个节点记录模型版本、Prompt 版本、输入输出摘要、耗时、成本和错误码。
- 对高风险动作使用「AI 建议 + 人工确认」，不要让模型直接执行。
- 工作流平台要支持暂停、恢复、取消、超时、重放和审计。

---

## 六、一句话总结

LLM 工作流编排的核心不是把模型调用排成链，而是把 AI 决策纳入可控的业务状态机：节点有契约，状态可恢复，失败可补偿，人工可介入，全链路可审计。
