# 多 Agent 协作与编排设计

## 核心概念

**多 Agent 系统**将复杂任务拆给多个具有不同职责、工具权限、上下文视角的 Agent 协作完成。面试重点不是「多个 Prompt 角色扮演」，而是**协议、状态、编排**——把多 Agent 当成分布式协作系统设计。

**常见角色：**

| 角色 | 职责 |
| --- | --- |
| Planner | 拆解任务、生成计划 |
| Researcher | 检索、收集证据 |
| Executor | 调用工具、执行动作 |
| Critic/Reviewer | 检查结果、提出修正 |
| Router/Coordinator | 任务路由与调度 |
| Summarizer | 压缩过程、生成终稿 |

**协作编排模式：**

- **顺序**：Planner → Executor → Reviewer
- **并行**：多个 Specialist 处理子任务后聚合
- **竞赛**：多候选 + Judge 选择
- **黑板**：共享任务板，Coordinator 调度
- **层级**：Manager 拆任务，Worker 执行

**Coordinator vs Judge：** Coordinator 回答「下一步谁做」（调度、超时、重试）；Judge 回答「哪个结果更可信」（质量、事实、风险）。复杂系统建议拆分，避免「自己分配、自己判对」的闭环偏差。

**Agent 通信协议**需结构化，而非自由对话：

```json
{
  "task_id": "task_123",
  "from": "researcher",
  "to": "reviewer",
  "message_type": "evidence_report",
  "content": {
    "claim": "应优先使用权限过滤后的检索结果",
    "evidence": ["doc_1", "doc_2"],
    "confidence": 0.82
  },
  "status": "completed",
  "next_action": "review"
}
```

生产原则：**关键流程由代码/工作流控制**，模型仅在语义判断、计划生成、内容生成节点发挥作用。

---

## 核心知识点

### 1. 何时适合 / 不适合多 Agent

**适合：** 任务可拆分；需多专业视角；需相互校验；子任务可并行；工具权限需隔离（只读 vs 写分离）。

**不适合：** 简单问答；边界不清；延迟极敏感；无评估机制硬堆 Agent；无权限/审计的高安全场景。

### 2. 架构模式

**Planner-Executor-Reviewer：** 职责清晰、易调试；风险是计划错误传导、Reviewer 只查语言不查执行、修复循环。

**Manager-Worker：** 多文件/多数据源并行；Worker 输入要窄；汇总保留来源引用；区分可跳过/可重试/必须完成的子任务。

**Debate/Self-Consistency：** 多候选+Judge；注意多模型可能共享同一错误上下文，Judge 可能偏向流畅而非正确，事实类仍需工具/规则。

**黑板模式：** 灵活但易状态膨胀、冲突结论、重复写入。

### 3. 任务拆分原则

按能力、权限、领域、数据源、阶段拆。每个 Agent 能答：**负责什么、不能做什么、输出交给谁**。职责描述过长通常说明边界未清。

### 4. 状态管理与上下文隔离

| 状态类型 | 内容 |
| --- | --- |
| Global State | 目标、用户约束、配置 |
| Agent Local | 中间推理、草稿（默认不共享） |
| Shared Artifacts | 证据卡片、结构化数据 |
| Decision Log | 决策、理由、责任 Agent、时间 |
| Execution Trace | 工具调用、错误、重试 |

**按角色最小必要上下文分发**，非广播全量历史。

### 5. 冲突仲裁

规则优先（安全/合规）> 证据优先 > 领域 Specialist > Judge 评分 > 人工确认。设置最大轮次、超时、置信度阈值，禁止无限争论。

### 6. 编排骨架示例

```python
from dataclasses import dataclass, field
from typing import Any, Literal

Role = Literal["planner", "executor", "reviewer"]

@dataclass
class TaskState:
    task_id: str
    user_goal: str
    max_iterations: int = 3
    artifacts: dict[str, Any] = field(default_factory=dict)

class MultiAgentOrchestrator:
    def run(self, state: TaskState) -> dict[str, Any]:
        plan = self.plan(state)
        for iteration in range(state.max_iterations):
            result = self.execute(state, plan)
            review = self.review(state, result)
            if review["approved"]:
                return {"status": "completed", "result": result, "iterations": iteration + 1}
            plan = self.revise_plan(plan, review)  # 中文注释：有限轮修复，防无限循环
        return {"status": "needs_human_review", "reason": "exceeded_max_iterations"}
```

### 7. 可靠性与观测

失败模式：循环调用、责任漂移、错误放大、上下文污染、成本失控、工具越权。

控制：每任务步数/token 预算；每 Agent 明确 I/O/权限/超时；Reviewer 检查**真实产物**非仅自然语言；全链路 trace。

指标：任务成功率、步骤成功率、冲突率、Reviewer 拦截率、人工介入率、token 成本、P95 延迟。

### 8. 与工作流关系

工作流管确定性状态机；多 Agent 管语义协作。**二者结合**：工作流兜关键路径，Agent 处理非结构化环节。

---

## 高频面试问题与标准答案

**1. 多 Agent 与单 Agent 最大区别？**  
单 Agent 链路短、成本低；多 Agent 拆角色、隔离权限、可校验，但带来通信、延迟、状态、仲裁复杂度。仅当收益大于成本时引入。

**2. 什么场景不该用？**  
简单问答、短改写、固定分类、低延迟接口；或无结构化消息、终止条件、评估的开放式多 Agent。

**3. 如何防无限循环？**  
编排层硬约束：最大迭代、工具/token 预算、每轮须有状态变化、连续两轮无新信息则终止、审查失败限轮修复、重复参数拦截。

**4. 输出冲突怎么办？**  
按类型：合规→规则；事实→证据；专业判断→Specialist/Judge；高风险→人工。保留来源、证据、置信度与 trace。

**5. Reviewer 如何有效？**  
明确检查项：证据、用户约束、安全规则、工具结果是否支持结论。检查 trace 与结构化产物，非只看 Executor 总结。

**6. 上下文如何共享？**  
最小必要：Coordinator 看全局；Worker 看子任务；Reviewer 看约束+产物+证据；Finalizer 看已批准结果。私有推理默认不进全局。

**7. 如何评估？**  
不只最终答案：计划正确性、工具准确率、Reviewer 拦截率、冲突率、人工介入率、轮次与成本。离线黄金集回放 trace。

**8. 权限控制？**  
权限绑定角色与工具，非仅写 Prompt。Researcher 只读；高风险写需权限系统+人工确认；记录调用者、租户、参数、结果。

**9. 最大线上风险？**  
复杂度失控：链路过长、状态难追踪、错误传播。优先小闭环：高价值场景、少量角色、固定流程、结构化输出、全链路 trace。

**10. 设计自动研究报告系统？**  
澄清输入输出与引用要求 → Coordinator/Researcher/Analyst/Writer/Reviewer 分工 → 证据卡片格式 → Reviewer 验引用 → 超时重试 → 高风险人工复核 → 评事实率与引用覆盖率。

---

## 面试回答加分点

1. **先问「是否真的需要多 Agent」**，再谈角色与编排。
2. **用研究报告/代码审查题**展示：需求澄清 → 分工 → 数据流 → 可靠性 → 指标。
3. **强调「流程系统兜底，Agent 负责语义」**，反对完全开放式自治。
4. **误区对照**：多 Prompt 扮演 ≠ 多 Agent；全量共享上下文；Reviewer 只改措辞；无 trace 无法排障。
5. **七步答题模板**：必要性 → 角色边界 → 编排模式 → 状态管理 → 冲突/终止 → 观测评估 → 风险（循环、污染、成本、权限）。
6. **一句话**：多 Agent 的价值是职责清晰、可并行、可审查、可控权限；生产要可追踪、可拦截、可回放。
