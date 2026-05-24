# AI 应用开发面试主题：多 Agent 协作与编排设计

## 主题选择记录

- 本次主题：多 Agent 协作与编排设计
- 所属分类：AI 应用开发 / Agent
- 已回顾历史主题：RAG、Agent 规划执行、Agent 工具调用、Prompt 工程、评估观测、安全防护、上下文工程、工作流编排、MCP、模型网关、多模态、语音 Agent、测试与发布治理
- 不重复原因：已有 Agent 主题重点讲单 Agent 的工具调用、规划执行和可靠性治理；本主题聚焦多个 Agent 如何分工、通信、共享状态、仲裁冲突、控制成本并稳定落地。
- 适用岗位：AI 应用开发工程师、Agent 平台工程师、LLM 应用架构师、AI Workflow 工程师、智能办公 / 数据分析 / 自动化任务系统工程师。

## 一、为什么这是高频考点

多 Agent 是面试中很容易被问深的主题，因为它看起来像“让多个模型角色协作”，实际考察的是候选人是否理解复杂 AI 应用的工程边界。

面试官通常不是想听“一个 Agent 当规划者，一个 Agent 当执行者”这种表层答案，而是会追问：

- 多 Agent 比单 Agent 好在哪里，什么时候反而更差？
- Agent 之间怎么通信，消息格式如何约束？
- 共享上下文怎么管理，如何避免上下文污染？
- 多个 Agent 给出冲突结论时谁说了算？
- 怎么避免循环调用、成本爆炸和责任不清？
- 线上怎么观测、回放、评估和调试？

好的回答要把多 Agent 当成一个分布式协作系统来设计，而不是简单堆叠多个 Prompt。

## 二、核心概念

### 1. 多 Agent 系统

多 Agent 系统是指将一个复杂任务拆给多个具有不同职责、工具权限、上下文视角或策略目标的 Agent 协作完成。

常见角色包括：

- Planner：拆解任务、生成计划、决定执行顺序。
- Researcher：检索资料、读取文档、收集证据。
- Executor：调用工具、修改数据、执行具体动作。
- Critic / Reviewer：检查结果、发现漏洞、提出修正建议。
- Router / Coordinator：判断任务应该交给哪个 Agent。
- Summarizer：压缩中间过程，生成最终输出。

面试中要强调：多 Agent 的核心不是“人设更多”，而是“职责边界更清晰，复杂任务可控拆解”。

### 2. 协作编排

协作编排是指控制多个 Agent 的执行顺序、依赖关系、状态流转和终止条件。

常见模式：

- 顺序编排：Planner -> Executor -> Reviewer。
- 并行编排：多个 Specialist 同时处理子任务，最后聚合。
- 竞赛编排：多个 Agent 生成候选答案，由 Judge 选择。
- 黑板模式：多个 Agent 读写共享任务状态，由 Coordinator 调度。
- 层级编排：Manager Agent 拆任务，Worker Agent 执行子任务。

生产系统中，编排逻辑通常不应完全交给模型自由决定。更稳妥的做法是：关键流程由代码或工作流引擎控制，模型只在语义判断、计划生成和内容生成节点发挥作用。

### 3. Agent 通信协议

Agent 通信协议定义 Agent 之间交换什么信息、使用什么结构、如何表达状态和错误。

一个可落地的消息结构通常包含：

```json
{
  "task_id": "task_123",
  "from": "researcher",
  "to": "reviewer",
  "message_type": "evidence_report",
  "content": {
    "claim": "需要优先使用权限过滤后的检索结果",
    "evidence": ["doc_1", "doc_2"],
    "confidence": 0.82
  },
  "status": "completed",
  "next_action": "review"
}
```

面试中不要只说“Agent 之间互相对话”。可靠系统要有结构化消息、任务 ID、状态、来源、置信度、错误码和可追踪日志。

### 4. Coordinator 与 Judge

Coordinator 负责调度任务，回答“下一步谁来做”；Judge 负责评价结果，回答“哪个结果更可信”。

两者职责不同：

- Coordinator 更像工作流调度器，关注任务分配、依赖关系、超时、重试、终止条件。
- Judge 更像评审器，关注答案质量、事实一致性、业务规则、风险等级。

小系统中可以让一个 Agent 同时承担协调和评审；复杂系统中建议拆开，否则容易出现“自己分配任务、自己判断正确”的闭环偏差。

## 三、核心知识点

### 1. 什么时候适合多 Agent

适合多 Agent 的场景：

- 任务天然可拆分：例如“调研竞品、分析差异、生成方案、审查风险”。
- 需要多种专业视角：例如法务、财务、技术、产品分别评审。
- 需要相互校验：生成 Agent 输出后，Review Agent 做事实核查和规则检查。
- 子任务可并行：多个数据源、多个文档、多个候选方案可以同时处理。
- 工具权限需要隔离：只读 Agent 和写操作 Agent 分离，降低误操作风险。

不适合多 Agent 的场景：

- 一次简单问答，用单模型调用即可完成。
- 任务边界不清，拆分后反而增加沟通成本。
- 对延迟极其敏感，无法承受多轮模型调用。
- 缺少评估机制，只是把多个 Agent 的输出拼起来。
- 安全要求很高，但没有权限控制、审计和人工确认。

面试回答可以用一句话概括：多 Agent 适合复杂、可拆、可校验的任务；不适合简单、低延迟、边界模糊的任务。

### 2. 常见架构模式

#### 模式一：Planner-Executor-Reviewer

这是最常见、也最容易落地的模式。

```text
用户目标
  -> Planner 拆解任务
  -> Executor 执行步骤
  -> Reviewer 检查结果
  -> Executor 修复问题
  -> Finalizer 生成最终答案
```

优点：

- 职责清晰，便于调试。
- Reviewer 可以拦截明显错误。
- 适合代码生成、数据分析、文档生成、办公自动化。

风险：

- Planner 计划错误会影响后续所有步骤。
- Reviewer 如果只做语言层面检查，可能发现不了真实执行错误。
- 多轮修复容易循环，需要最大迭代次数和终止条件。

#### 模式二：Manager-Worker

Manager 负责拆解和分配任务，多个 Worker 处理子任务，最后汇总。

适合场景：

- 多文件分析。
- 多数据源调研。
- 多模块代码审查。
- 大任务分片处理。

关键点：

- Worker 的输入必须足够窄，避免每个 Worker 都看到全量上下文。
- Manager 汇总时要保留来源引用，避免把子结论混成不可追踪的最终答案。
- Worker 失败不能直接导致全局失败，应区分可跳过、可重试和必须完成的子任务。

#### 模式三：Debate / Self-Consistency

多个 Agent 从不同角度生成候选答案，再由 Judge 评估。

适合场景：

- 复杂推理。
- 方案比较。
- 需求评审。
- 风险识别。

注意点：

- Debate 不等于真理发现。多个模型都可能受同一错误上下文影响。
- Judge 也可能偏向更流畅的答案，而不是更正确的答案。
- 对事实类问题，最终仍要依赖外部证据、工具结果或规则校验。

#### 模式四：黑板模式

所有 Agent 围绕一个共享任务状态工作，读取当前状态，写入自己的发现、动作和建议。

```text
Shared Task Board
  - 用户目标
  - 当前计划
  - 已完成步骤
  - 证据列表
  - 待解决问题
  - 风险项
  - 最终结论草稿
```

优点是灵活，适合探索式任务；缺点是状态治理难，容易出现重复写入、冲突结论和上下文膨胀。

### 3. 任务拆分原则

任务拆分不是越细越好。拆得太粗，Agent 职责不清；拆得太细，通信成本和模型调用成本会上升。

实用拆分原则：

- 按能力拆：检索、计算、生成、审核、执行分别拆开。
- 按权限拆：只读、低风险写入、高风险写入分离。
- 按领域拆：技术、产品、合规、财务分别处理。
- 按数据源拆：不同知识库、不同系统由不同 Worker 读取。
- 按阶段拆：计划、执行、校验、总结分阶段推进。

每个 Agent 都应该能回答三个问题：

1. 我负责什么？
2. 我不能做什么？
3. 我的输出交给谁消费？

如果一个 Agent 的职责描述需要很长，通常说明边界没有拆清楚。

### 4. 状态管理与上下文隔离

多 Agent 最大的工程难点之一是状态管理。

常见状态类型：

- Global State：任务目标、用户约束、全局配置。
- Agent Local State：某个 Agent 的中间推理、临时草稿。
- Shared Artifacts：文档、代码、检索结果、结构化数据。
- Decision Log：关键决策、理由、责任 Agent、时间点。
- Execution Trace：工具调用、输入输出、错误和重试记录。

推荐做法：

- 全局状态只放稳定事实和明确约束，不放所有中间推理。
- Agent 本地推理不默认共享，避免污染其他 Agent。
- 共享产物使用结构化格式，例如 JSON、Markdown section、数据库记录。
- 关键决策写入 Decision Log，方便回放和审计。
- 长任务要做上下文压缩，但压缩结果必须保留证据来源。

面试中可以强调：多 Agent 不是把所有对话历史塞给每个 Agent，而是按角色最小必要上下文分发。

### 5. 冲突处理与仲裁机制

多个 Agent 给出不同结论很常见。系统必须有明确的冲突处理策略。

常见策略：

- 规则优先：安全、权限、合规规则优先级高于模型判断。
- 证据优先：带可靠证据来源的结论优先于无来源结论。
- 权威 Agent 优先：特定领域交给特定 Specialist 裁决。
- Judge 评分：按事实一致性、完整性、风险、可执行性评分。
- 人工确认：高风险动作或无法自动裁决时进入 Human-in-the-loop。

不要让多个 Agent 无限争论。生产系统要设置最大轮次、超时、置信度阈值和降级策略。

### 6. 可靠性控制

多 Agent 系统的失败模式比单 Agent 更多。

常见失败：

- 循环调用：Agent 互相等待或反复修改同一问题。
- 责任漂移：每个 Agent 都以为别人会处理关键步骤。
- 错误放大：上游错误计划被下游执行并包装成可信结论。
- 上下文污染：某个 Agent 的错误假设被共享后影响所有 Agent。
- 成本失控：一次用户请求触发大量模型调用。
- 工具误用：低权限 Agent 绕过边界触发高风险工具。

工程控制手段：

- 每个任务设置最大步骤数、最大模型调用数和预算。
- 每个 Agent 有明确输入、输出、权限和超时。
- 高风险工具调用必须经过权限校验和人工确认。
- Reviewer 必须检查真实产物，而不是只检查自然语言描述。
- 关键节点保存 trace，支持失败回放。
- 对常见错误建立自动化评测集和回归测试。

### 7. 观测与评估

多 Agent 系统不能只看最终答案是否生成，还要观察协作过程。

核心指标：

- Task Success Rate：任务最终成功率。
- Step Success Rate：每个步骤成功率。
- Tool Error Rate：工具调用错误率。
- Iteration Count：平均协作轮次。
- Token Cost：单任务 token 成本。
- Latency P50 / P95：端到端延迟。
- Human Escalation Rate：人工介入比例。
- Conflict Rate：Agent 结论冲突比例。
- Reviewer Catch Rate：Reviewer 发现问题的比例。
- False Approval Rate：Reviewer 错误放行比例。

面试加分点：不仅要评估最终输出，还要评估计划质量、工具调用正确性、审查有效性和成本延迟。

## 四、工程实现示例

下面是一个简化的多 Agent 编排骨架，重点展示职责分离、结构化消息、预算控制和审查闭环。

```python
from dataclasses import dataclass, field
from typing import Any, Literal


Role = Literal["planner", "executor", "reviewer"]


@dataclass
class AgentMessage:
    task_id: str
    sender: Role
    receiver: Role
    message_type: str
    content: dict[str, Any]
    status: Literal["pending", "completed", "failed"]


@dataclass
class TaskState:
    task_id: str
    user_goal: str
    budget_tokens: int
    max_iterations: int = 3
    messages: list[AgentMessage] = field(default_factory=list)
    artifacts: dict[str, Any] = field(default_factory=dict)


class MultiAgentOrchestrator:
    def run(self, state: TaskState) -> dict[str, Any]:
        plan = self.plan(state)
        state.artifacts["plan"] = plan

        for iteration in range(state.max_iterations):
            result = self.execute(state, plan)
            state.artifacts["latest_result"] = result

            review = self.review(state, result)
            state.artifacts["latest_review"] = review

            if review["approved"]:
                return {
                    "status": "completed",
                    "result": result,
                    "review": review,
                    "iterations": iteration + 1,
                }

            # 中文注释：只允许有限轮修复，避免 Agent 在审查和修复之间无限循环。
            plan = self.revise_plan(plan, review)

        return {
            "status": "needs_human_review",
            "reason": "exceeded_max_iterations",
            "last_result": state.artifacts.get("latest_result"),
            "last_review": state.artifacts.get("latest_review"),
        }

    def plan(self, state: TaskState) -> list[dict[str, str]]:
        return [
            {"step": "collect_context", "owner": "executor"},
            {"step": "produce_answer", "owner": "executor"},
            {"step": "verify_answer", "owner": "reviewer"},
        ]

    def execute(self, state: TaskState, plan: list[dict[str, str]]) -> dict[str, Any]:
        return {
            "answer": f"围绕目标生成结果：{state.user_goal}",
            "evidence": ["artifact://context"],
        }

    def review(self, state: TaskState, result: dict[str, Any]) -> dict[str, Any]:
        has_evidence = bool(result.get("evidence"))
        return {
            "approved": has_evidence,
            "issues": [] if has_evidence else ["缺少证据来源"],
            "risk_level": "low" if has_evidence else "medium",
        }

    def revise_plan(
        self,
        plan: list[dict[str, str]],
        review: dict[str, Any],
    ) -> list[dict[str, str]]:
        return [{"step": "fix_review_issues", "owner": "executor"}, *plan]
```

这个示例的面试价值不在于代码复杂，而在于体现四个工程要点：

1. 编排器由代码控制，不完全依赖模型自由循环。
2. Planner、Executor、Reviewer 的职责分离。
3. 最大迭代次数是硬约束。
4. 审查失败后可以修复，但超过阈值要进入人工处理或降级。

## 五、典型系统设计题回答框架

如果面试题是：“设计一个多 Agent 自动研究报告系统”，可以按下面结构回答。

### 1. 需求澄清

- 输入是什么：主题、文档、网页、数据库还是内部知识库？
- 输出是什么：摘要、对比分析、完整报告还是可执行方案？
- 是否需要引用来源？
- 是否允许联网和调用内部工具？
- 延迟和成本约束是什么？
- 哪些内容需要人工确认？

### 2. Agent 分工

- Coordinator：拆解任务和控制流程。
- Researcher：检索资料并产出证据卡片。
- Analyst：基于证据做结构化分析。
- Writer：生成报告草稿。
- Reviewer：检查事实、引用、逻辑和风险。

### 3. 状态与数据流

```text
用户主题
  -> Coordinator 生成任务计划
  -> Researcher 并行检索资料
  -> Analyst 结构化归纳
  -> Writer 生成报告
  -> Reviewer 检查引用和逻辑
  -> Finalizer 输出最终报告
```

### 4. 可靠性设计

- 所有结论必须绑定证据来源。
- Researcher 输出统一证据卡片格式。
- Reviewer 检查引用是否真实存在。
- 每个阶段有超时和最大重试次数。
- 高风险结论进入人工复核。
- trace 全量记录，支持失败回放。

### 5. 评估指标

- 报告事实准确率。
- 引用覆盖率。
- 引用无效率。
- 用户采纳率。
- 平均生成成本。
- 端到端延迟。
- Reviewer 拦截问题数。

这样回答会比“多个 Agent 分别写不同章节”更有工程深度。

## 六、高频面试问题与参考答案

### Q1：多 Agent 和单 Agent 最大区别是什么？

**参考答案：**

单 Agent 通常由一个模型实例完成理解、规划、执行和总结，优点是链路短、成本低、调试简单。多 Agent 会把复杂任务拆给多个角色，每个角色有不同上下文、工具权限和输出协议，适合复杂、可拆、需要相互校验的任务。

但多 Agent 不是天然更强。它会带来通信成本、延迟、状态管理、冲突仲裁和观测调试复杂度。生产环境只有在任务复杂度、权限隔离或质量校验收益大于这些成本时，才值得引入多 Agent。

### Q2：什么场景不应该使用多 Agent？

**参考答案：**

简单问答、短文本改写、固定流程分类、低延迟接口通常不适合多 Agent。因为单次模型调用或普通工作流就能解决，引入多 Agent 只会增加成本和不确定性。

另外，如果系统没有结构化消息、状态管理、评估指标和终止条件，也不应该直接做开放式多 Agent。否则很容易出现循环调用、错误互相放大和线上不可调试的问题。

### Q3：多 Agent 如何防止无限循环？

**参考答案：**

需要从编排层设置硬约束，而不是只靠 Prompt 让模型“不要循环”。

常见做法包括：

- 设置最大迭代次数。
- 设置最大工具调用次数和 token 预算。
- 每轮必须产生结构化状态变化。
- 如果连续两轮没有新增信息，就终止或转人工。
- 审查失败只允许进入有限轮修复。
- 对相同工具参数做幂等和重复调用拦截。

核心原则是：终止条件由系统控制，模型只能建议下一步，不能无限自由调度自己。

### Q4：多个 Agent 输出冲突时怎么办？

**参考答案：**

先看冲突类型。如果是安全、权限、合规类冲突，规则优先；如果是事实类冲突，证据优先；如果是专业判断冲突，可以交给领域 Specialist 或 Judge；如果风险高或置信度不足，应进入人工确认。

工程上要保留每个结论的来源、证据、置信度和生成上下文，不能只保留最终答案。没有 trace 的冲突仲裁很难复盘，也很难持续优化。

### Q5：Reviewer Agent 怎么设计才有效？

**参考答案：**

Reviewer 不能只是“看看答案好不好”。有效的 Reviewer 应该有明确检查项，例如事实是否有证据、是否满足用户约束、是否违反安全规则、工具执行结果是否支持结论、输出格式是否可解析。

更可靠的做法是让 Reviewer 检查真实产物和结构化 trace，而不是只看 Executor 的自然语言总结。对于高风险任务，Reviewer 只能给风险评估和建议，最终写操作仍需要权限系统或人工确认。

### Q6：多 Agent 的上下文应该怎么共享？

**参考答案：**

不要把所有历史消息广播给每个 Agent。推荐按角色分发最小必要上下文：

- Coordinator 看到全局目标、计划和状态。
- Worker 只看到自己的子任务、必要输入和输出格式。
- Reviewer 看到用户约束、产物、证据和执行 trace。
- Finalizer 看到被批准的结构化结果。

共享内容应该以结构化 artifact 为主，例如证据卡片、计划表、执行日志、审查报告。Agent 的私有推理过程不应默认进入全局上下文，避免错误假设污染后续环节。

### Q7：如何评估多 Agent 系统效果？

**参考答案：**

不能只评估最终答案。要同时评估任务成功率、计划正确性、工具调用正确率、Reviewer 拦截率、冲突率、人工介入率、平均轮次、延迟和成本。

离线评测可以使用黄金任务集，回放完整 trace，检查每一步是否符合预期。线上评估要采集用户反馈、失败原因、Agent 角色、工具错误和成本数据。多 Agent 的优化重点通常不是“换更强模型”，而是定位哪个角色、哪个步骤、哪个协议导致失败。

### Q8：多 Agent 和工作流编排有什么关系？

**参考答案：**

工作流编排更强调确定性的状态机和流程控制，多 Agent 更强调模型角色的语义协作。生产系统通常会把二者结合：用工作流控制关键路径、权限、重试、超时和状态流转；用 Agent 处理计划生成、语义判断、内容生成、异常分析等非结构化环节。

换句话说，工作流提供边界和可控性，Agent 提供灵活性。完全开放式多 Agent 更适合实验和低风险场景，高价值生产场景通常需要工作流兜住关键链路。

### Q9：多 Agent 如何做权限控制？

**参考答案：**

权限要绑定 Agent 角色和工具，不应只写在 Prompt 里。可以把工具分为只读、低风险写入、高风险写入。Researcher 只能读数据，Executor 可以执行低风险操作，高风险操作必须由权限系统校验并进入人工确认。

每次工具调用都要记录调用者、用户身份、租户、参数、结果和风险等级。即使某个 Agent 通过 Prompt 生成了越权调用请求，应用层也必须拦截。

### Q10：多 Agent 最大的线上风险是什么？

**参考答案：**

最大的风险是复杂度失控：调用链变长、状态难追踪、成本不可预测、错误在多个 Agent 之间传播。表面上看是“多个专家协作”，实际上如果没有协议、日志、预算、终止条件和评估，就会变成不可调试的黑盒系统。

所以落地多 Agent 时应优先做小闭环：明确一个高价值场景、少量角色、固定流程、结构化输出、全链路 trace 和离线评测。不要一开始就做完全开放式自治系统。

## 七、面试答题模板

回答多 Agent 问题时，可以按下面模板组织：

1. 先判断是否真的需要多 Agent：任务是否复杂、可拆、可校验。
2. 定义角色边界：每个 Agent 的职责、输入、输出、工具权限。
3. 设计编排方式：顺序、并行、Manager-Worker、Reviewer 闭环。
4. 设计状态管理：全局状态、本地产物、共享 artifact、决策日志。
5. 设计冲突与终止：Judge、规则优先、最大轮次、人工介入。
6. 设计观测评估：trace、成功率、成本、延迟、冲突率、审查有效性。
7. 说明风险：循环、上下文污染、成本爆炸、权限越界和调试困难。

## 八、常见误区

- 误区一：以为多 Agent 等于多个 Prompt 角色扮演。真正关键是协议、状态和编排。
- 误区二：所有 Agent 共享完整上下文。这样容易污染上下文并增加成本。
- 误区三：让模型自己决定无限下一步。生产系统必须有硬终止条件。
- 误区四：Reviewer 只检查文字表达。Reviewer 应该检查证据、工具结果和业务规则。
- 误区五：没有 trace。多 Agent 没有 trace 基本无法定位线上问题。
- 误区六：忽视权限。Agent 角色不能绕过真实用户权限和业务风控。

## 九、一分钟总结

多 Agent 的价值在于把复杂 AI 任务拆成职责清晰、可并行、可审查、可控权限的协作流程。面试中不要只讲“多个 Agent 互相讨论”，要讲清楚角色边界、通信协议、状态管理、冲突仲裁、终止条件、权限控制、观测评估和成本治理。生产落地的核心原则是：流程由系统兜底，Agent 负责语义能力；关键动作可追踪，风险动作可拦截，失败过程可回放。
