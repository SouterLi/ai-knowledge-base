# LLM 工作流编排与 Human-in-the-loop

## 核心概念

LLM 进入真实业务后通常是**可控多步流程**，而非「问一句答一句」：

```text
输入校验 → 意图识别 → 检索/工具 → 模型生成 → 规则校验 → 人工审核 → 发布 → 审计归档
```

**工作流编排**把多个模型调用、规则节点、工具节点和人工节点组织成可观测、可恢复、可治理的业务流程。

**三类系统对比：**

| 类型 | 核心目标 | 典型场景 |
| --- | --- | --- |
| 普通 API | 同步返回 | 简单问答、翻译 |
| 异步任务 | 长任务后台执行 | 报告生成、批量分析 |
| 工作流编排 | 多节点、分支、状态、可暂停恢复 | 合同审查、工单、审批助手 |

**Human-in-the-loop（HITL）** 不是「模型不行就找人」，而是风险治理：低置信、高风险、合规敏感、规则冲突时暂停流程，由人决策后恢复。

**面试金句：** LLM 工作流不是把多个 Prompt 串起来，而是把 AI 能力放进**可审计、可回滚、可人工接管**的业务状态机。

---

## 核心知识点

### 1. 工作流六要素

Trigger（API/定时/MQ）→ Step/Node → State（`pending/running/waiting_for_review/succeeded/failed`）→ Transition（分支条件）→ Context（跨节点数据）→ Policy（权限、预算、重试、审计）。

### 2. DAG、状态机与 Agent

| 维度 | DAG | 状态机 | Agent |
| --- | --- | --- | --- |
| 流程 | 节点边相对固定 | 事件驱动迁移 | 模型动态决定 |
| 可控性 | 高 | 很高 | 中等 |
| 场景 | 内容生产固定链路 | 工单/审批生命周期 | 开放探索 |
| 组合 | 外层状态机 + 内层 DAG + 局部 Agent 子流程 | | |

### 3. 节点契约

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
    category = "refund" if "退款" in ticket["content"] else "general"
    if ticket.get("vip") and category == "refund":
        # 中文注释：VIP 退款走人工，非模型自主决定
        return WorkflowStepResult(
            status="need_human",
            output={"category": category},
            reason="VIP 退款需人工确认",
        )
    return WorkflowStepResult(status="ok", output={"category": category})
```

每节点需：输入/输出 Schema、错误分类、超时与 token 预算、审计字段（模型/Prompt/工具版本）。

### 4. HITL 触发与审核字段

触发：低置信、退款/发邮件/改权限、合规领域、模型与规则冲突、用户升级、循环/超预算。

审核记录：`workflow_id`、`step_id`、`review_reason`、`ai_suggestion`、`evidence`、`reviewer_id`、`decision`（approve/reject/edit/return）、`decision_note`。

### 5. 暂停、恢复与持久化

```sql
-- 中文注释：状态落库，服务重启或跨天审核不丢上下文
CREATE TABLE workflow_runs (
  id TEXT PRIMARY KEY,
  workflow_name TEXT NOT NULL,
  status TEXT NOT NULL,
  current_step TEXT NOT NULL,
  context_json JSON NOT NULL,
  idempotency_key TEXT,
  updated_at TIMESTAMP NOT NULL
);
```

恢复时按 `current_step` + `context_json` 继续，非从头重跑。

### 6. 重试、补偿、幂等

- **可重试**：网络超时、429、临时工具不可用
- **不重试**：参数非法、权限不足、合规拦截
- **补偿**：已有副作用时撤销或修正
- **幂等**：同一 `request_id` 不重复退款/建单

```python
def execute_refund_step(order_id: str, amount: int, request_id: str) -> dict:
    existing = find_refund_by_request_id(request_id)
    if existing:
        return existing  # 中文注释：重试命中幂等，直接返回
    return create_refund(order_id, amount, request_id=request_id)
```

**模型节点可重试；有副作用节点先幂等再重试。**

### 7. 上下文分层

业务上下文（订单号、审核结论）必须结构化；模型上下文（Prompt、检索、草稿）用于追溯；临时变量可过期；审计上下文永久可查。每节点只读最小必要字段。

### 8. 权限与审计

用户能否触发 workflow；节点能否调某工具；审核员数据权限；模型建议不可绕过业务权限。**模型不能绕过权限，人工也不能绕过数据权限。**

### 9. 典型架构

```text
Client → Workflow Gateway（鉴权/幂等）→ Workflow DB
      → Queue/Scheduler → Worker/Orchestrator
      → LLM / Retrieval / Tools / Rule Engine
      → Human Review Console → Audit / Metrics / Trace
```

### 10. 防死循环

最大步数、重试次数、token/运行时间预算；Agent 子流程限制工具集；同类错误熔断；连续失败进人工或失败终态。

---

## 高频面试问题与标准答案

**1. 工作流与普通异步任务区别？**  
异步解决「耗时长」；工作流解决「多步骤决策」——分支、状态迁移、人工审核、节点重试、补偿、审计。二者常 combined。

**2. 何时需要 HITL？**  
影响真实权益、合规、体验时：低置信、高风险操作、规则冲突、敏感领域、用户升级、连续失败。

**3. 人工审核节点怎么设计？**  
进入 `waiting_for_review` 后持久化上下文；审核台展示建议、证据、触发原因；提交 decision 后记录审核人并恢复流转；全程可追溯。

**4. 执行到一半失败？**  
区分错误类型：临时→退避重试；参数/权限/合规→失败或人工；有副作用→幂等+补偿；从最近成功节点或等待节点恢复。

**5. 为何不能只用 Prompt 串联？**  
缺状态、难恢复、难控权限、难审计、难统计成本。上线需显式节点+契约+日志。

**6. DAG 还是状态机？**  
固定依赖链用 DAG；明确业务生命周期（open→pending_review→resolved）用状态机；复杂系统外层状态机+内层 DAG。

**7. 高风险工具能否模型自动调？**  
模型可**建议**；执行前应用层权限、参数、风控、人工确认；越高风险越要审批+幂等+审计。

**8. 如何测试？**  
节点单测（Schema、幂等）；流程测（分支、失败、人工恢复）；回归评测；混沌（超时、重复投递、Worker 重启）；权限测。

**9. 工作流上下文保存什么？**  
业务状态、节点 I/O、模型/Prompt 版本、检索证据、工具摘要、审核结论、错误信息；敏感字段脱敏。

**10. 与 Agent 文档边界？**  
本文：业务流程编排与 HITL；Agent 文档：工具调用与规划可靠性；不重复展开 Tool Schema。

---

## 面试回答加分点

1. **先画状态机再标 LLM 节点**，体现「业务优先、AI 嵌入」。
2. **强调暂停/恢复**与 `waiting_for_review`，区别于无状态 API。
3. **幂等+补偿成对出现**，回答副作用类追问。
4. **审核台三件套**：AI 建议 + 证据 + 触发原因，决策四态 approve/reject/edit/return。
5. **反对只存最终答案**；节点级 I/O 才能定位分类错还是生成错。
6. **一句话收束**：节点有契约，状态可恢复，失败可补偿，人工可介入，全链路可审计。
