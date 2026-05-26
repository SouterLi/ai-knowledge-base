# AI Agent 规划执行与可靠性治理

## 核心概念

**AI Agent** 是以大模型为推理核心，通过工具调用、任务规划、状态管理和执行反馈完成复杂目标的 AI 应用形态。与普通 Chatbot「一次输入、一次生成」不同，Agent 引入**循环式执行**：理解目标 → 规划步骤 → 选择工具 → 调用外部系统 → 读取结果 → 修正计划 → 输出答案。

**与普通 LLM 应用的区别：**

| 维度 | 普通 LLM 应用 | Agent |
| --- | --- | --- |
| 执行模式 | 单次生成 | 多步循环 |
| 外部能力 | 无或仅 RAG | 工具/API/代码执行 |
| 状态 | 上下文拼接 | 会话/任务/长期记忆 |
| 风险 | 内容质量 | 越权执行、真实副作用 |

**规划执行模式：**

- **ReAct**：推理（Reason）与行动（Act）交替，灵活但易循环、成本不可控。
- **Plan-and-Execute**：先整体计划再逐步执行，适合步骤明确的复杂任务，计划需动态修正。
- **Workflow Agent**：关键路径由代码/工作流固定，模型仅在特定节点做判断，**生产首选**。
- **Multi-Agent**：角色分工（规划者/执行者/审查者），增加通信与一致性成本。

**可靠性治理核心原则：** 模型只负责「建议调用什么」，真实执行权在受控应用层；高风险动作必须人工确认；全流程可观测、可回放。

```python
# 带预算与终止条件的 Agent 循环骨架
MAX_STEPS = 8

async def agent_loop(goal: str, tools: list, registry) -> dict:
    messages = [{"role": "user", "content": goal}]
    for step in range(MAX_STEPS):
        resp = await llm.chat(messages=messages, tools=tools)
        if not resp.tool_calls:
            return {"status": "done", "answer": resp.content, "steps": step + 1}
        # 中文注释：无实质进展时提前终止，避免空转
        if is_no_progress(messages, resp):
            return {"status": "stalled", "reason": "no_progress"}
        for tc in resp.tool_calls:
            result = await registry.execute(tc.name, tc.args)  # 应用层校验+执行
            messages.append(tool_result_message(tc.id, result))
    return {"status": "max_steps", "reason": "budget_exceeded"}
```

---

## 核心知识点

### 1. 记忆与状态分层

| 类型 | 内容 | 注意点 |
| --- | --- | --- |
| 会话状态 | 当前对话、任务进度、工具结果 | 按 token 预算裁剪 |
| 短期记忆 | 本轮任务临时事实 | 任务结束可丢弃 |
| 长期记忆 | 用户偏好、稳定知识 | 写入前需规则+授权，防污染 |
| 外部状态 | 订单/工单/DB | 以结构化字段引用，不全量塞上下文 |

### 2. 可靠性治理手段

- **工具白名单**：仅注册授权工具
- **参数校验**：Schema + 业务规则（如 order_id 归属 session）
- **权限分层**：只读 / 低风险写 / 高风险写
- **人工确认**：付款、删除、发外部消息、改生产数据
- **调用预算**：最大步数、token、工具次数、耗时
- **幂等设计**：写操作带幂等键，防重复执行
- **可观测性**：输入、计划、工具调用、结果、错误、成本
- **回滚/补偿**：高风险操作有撤销方案

### 3. 防无限循环

```python
def should_stop(state) -> bool:
    # 中文注释：连续两轮相同工具+相同参数 → 去重终止
    if state.repeated_call_count >= 2:
        return True
    if state.elapsed_ms > state.max_ms or state.tool_calls >= state.max_tools:
        return True
    if state.rounds_without_new_facts >= 2:
        return True
    return False
```

### 4. 评估指标（不能只看最终文案）

| 指标 | 含义 |
| --- | --- |
| 任务成功率 | 是否完成用户目标 |
| 工具选择准确率 | 是否选对工具 |
| 参数准确率 | 参数合法且符合意图 |
| 步骤效率 | 调用次数与 token 成本 |
| 错误恢复率 | 失败后能否修正 |
| 安全违规率 | 越权、泄漏、危险操作 |
| 可复现率 | 固定环境是否稳定 |

### 5. Prompt Injection 在 Agent 场景更危险

普通注入影响回答；Agent 场景可能诱导**调用工具、泄露数据、执行危险动作**。防护：不信任外部内容、隔离系统指令与 untrusted 数据、工具权限最小化、敏感操作二次确认、应用层策略校验。

### 6. 预览 + 确认两阶段

```python
# 中文注释：submit 仅在后端收到用户 confirm_token 后触发，不由模型自主调用
async def submit_refund(order_id: str, confirm_token: str, session) -> dict:
    if not verify_confirm_token(confirm_token, session.user_id, order_id):
        return {"error": "permission_denied", "message": "缺少用户确认"}
    return await refund_api.create(order_id, idempotency_key=confirm_token)
```

---

## 高频面试问题与标准答案

**1. 什么是 AI Agent？与 Chatbot 区别？**  
Agent 能围绕目标规划、调用工具、读取反馈并迭代执行；具备行动能力与状态驱动的多步执行，因此更需要权限、安全与可观测性治理。

**2. ReAct 优缺点？**  
优点：灵活，适合探索式任务。缺点：易循环、步骤漂移、成本不可控。生产需配合最大步数、工具白名单、失败退出。

**3. 何时选 Workflow 而非开放式 Agent？**  
业务流程稳定、风险高、需强一致性时（审批、支付、工单流转）。关键路径由代码控制，LLM 仅做意图识别、摘要、字段补全。

**4. 如何避免无限循环？**  
最大步数/工具次数/耗时/token 预算；重复调用去重；连续失败要求总结并退出；编排层检测无实质进展则中止或转人工。

**5. 工具调用失败如何处理？**  
分类：参数错误→结构化错误回灌改参；权限错误→终止；临时故障→有限重试；业务错误→明确原因；不可恢复→透明说明+降级/人工。

**6. Agent 记忆如何设计？**  
分层：会话状态、短期、长期。长期记忆不能自动写入全部内容，需写入规则、可见性、删除机制、敏感信息过滤。

**7. 如何评估可上线？**  
任务集+回放环境：成功率、工具/参数准确率、错误恢复、安全违规、成本延迟；红队测试覆盖越权、注入、恶意参数。

**8. Multi-Agent 一定更好吗？**  
不一定。角色分工可提升复杂任务组织性，但增加通信成本、延迟、一致性难度。先验证单 Agent 或 Workflow 是否足够。

**9. 可观测性应记录什么？**  
用户输入、Prompt/模型版本、工具候选、调用参数与结果、错误类型、耗时、token、最终输出、用户反馈；写操作还需操作者、确认记录、幂等键。

**10. 设计「自动处理退款咨询」Agent？**  
采用 Workflow：意图识别→查订单与规则→判断是否满足→生成解释；真实退款需人工/用户确认→调用退款 API→审计日志。LLM 负责语义，规则与执行由确定性服务完成。

---

## 面试回答加分点

1. **先定风险等级再选模式**：低风险探索用 ReAct；高价值生产用 Workflow + 局部 LLM。
2. **强调「模型建议、系统执行」**：三条铁律——模型只决策、参数不可信、工具即契约。
3. **用具体数字**：如 MAX_STEPS=8、工具 5～15 个/轮、P95 延迟与单任务 token 上限。
4. **对比 RAG**：RAG 解决「外部知识怎么检索」；Agent 解决「如何调用工具、拆解任务、可控执行」。
5. **设计题七步模板**：目标与风险 → 模式选择 → 工具/Schema/权限 → 状态与上下文 → 安全边界 → 评估指标 → 成本与降级。
6. **一句话收束**：Agent 核心不是让模型自由行动，而是用工具协议、编排、权限和评估让模型在可控边界内完成任务。
