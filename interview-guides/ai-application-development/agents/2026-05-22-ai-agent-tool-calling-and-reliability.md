# AI Agent：工具调用、规划执行与可靠性治理

## 主题选择记录

- **主题**：AI Agent 应用开发——工具调用、规划执行与可靠性治理
- **分类**：AI 应用开发 / Agents（体系化导读）
- **与仓库其他主题关系**：RAG 解决「外部知识怎么检索」；`interview-notes/agents` 侧重工具 Schema；本文件覆盖**编排模式、状态、评估与上线**
- **适用岗位**：AI 应用开发、Agent 平台工程师

## 核心概念

Agent 是以 LLM 为推理核心，通过**工具、规划、状态、反馈**完成复杂目标的系统，而不是单轮 Chatbot。

与普通 LLM 应用的区别：存在**多步循环**、**副作用**（写库、发邮件、退款）、**状态机**和**更高安全要求**。生产里的共识往往是：**不要把核心业务全交给开放式循环**——用 Workflow 固定高风险路径，在语义判断节点引入 LLM。

常见模式：

| 模式 | 特点 | 适用 |
|------|------|------|
| ReAct | 推理与行动交替，灵活 | 探索型、只读为主 |
| Plan-and-Execute | 先计划再执行 | 步骤多、可审计 |
| Workflow Agent | 代码控主路径 | 支付、审批、工单 |
| Multi-Agent | 角色分工 | 复杂但成本高 |

## 核心知识点

### 1. 工具调用全链路

用户目标 → 模型生成 `tool_call` → **解析/Schema/权限/预算** → 执行 → 结构化 `observation` → 继续或结束。模型永不直连生产 API。

```python
async def run_agent_turn(state, tools_for_intent):
    resp = await llm.chat(state.messages, tools=tools_for_intent)
    if not resp.tool_calls:
        return resp.content
    for call in resp.tool_calls:
        args = validate(call.name, call.arguments)  # 不信任模型
        if not authorize(state.user, call.name, args):
            obs = {"error": "FORBIDDEN"}
        else:
            obs = await execute_tool(call.name, args)
        state.messages.append(tool_message(call.id, summarize(obs)))
    if state.step >= state.max_steps:
        raise BudgetExceeded()
    return await run_agent_turn(state, tools_for_intent)
```

### 2. 工具 Schema 设计要点

名称单一职责；描述含「何时不用」；参数枚举化；返回 JSON；错误码分类；高风险标 `require_confirm` 和幂等键 `idempotency_key`。

### 3. ReAct 的坑与治理

易循环、步骤漂移、成本不可控。治理：`max_steps`、重复调用检测、无进展退出、只读先行、工具子集路由。

### 4. Plan-and-Execute

适合报告生成、多系统查询。计划可能错，需根据 observation **动态修订**；计划本身可记录审计。

### 5. Workflow vs 开放式 Agent

退款、改权限、删数据：**Workflow + LLM 填槽/解释**；开放式适合调研、草稿、只读分析。面试设计题先问**风险等级**再选模式。

### 6. 记忆与状态（融入 Agent 上下文）

- 会话状态：当前步骤、待确认操作。  
- 短期：本轮工具结果摘要。  
- 长期：经确认偏好（写入门槛）。  
工具原始结果存档，给模型看摘要即可。

### 7. 可靠性治理清单

工具白名单；参数校验；读写分级；人工确认（preview/submit）；调用预算（步数/token/费用/时间）；写操作幂等；全链路 trace；高风险补偿/回滚方案。

### 8. Multi-Agent 取舍

规划者/执行者/审查者可提高复杂任务组织性，但增加延迟、一致性和调试成本。先证明单 Agent 或 Workflow 不够再加。

### 9. 评估指标

任务成功率、工具选择准确率、参数准确率、步骤效率、错误恢复率、安全违规率、确认命中率、可复现率。

### 10. Prompt Injection 在 Agent 场景

外部文档可能诱导调用 `refund` 或泄露数据。隔离不可信内容；工具层硬权限；写操作必须人确认。

## 高频面试问题与标准答案

### 1. 什么是 AI Agent？和 Chatbot 有何不同？

Agent 会围绕目标多步调工具、读反馈、再决策，能改变外部系统状态。Chatbot 主要是生成文本。所以 Agent 面试一定会追权限、幂等、审计，而不只是 Prompt。

### 2. 为什么不能让模型直接执行工具？

输出不可信，注入可诱导危险调用。我会说：模型只提议，应用层有注册表、校验、鉴权、日志和预算——这是工程底线。

### 3. 如何设计工具 Schema？

少而精、描述清晰、参数强类型、返回结构化。两个工具别都叫 `search` 却一个查订单一个查人。退款类单独成高风险工具，走两阶段确认。

### 4. ReAct 优缺点？

灵活，适合开放探索；但容易死循环、成本高、中间推理不稳定。生产要加步数上限、重复检测，重要业务用 Workflow 包住。

### 5. Plan-and-Execute 适合什么？

步骤清晰的长任务，比如「拉三个系统数据写周报」。计划可展示给用户；执行中根据错误修订计划，而不是一次计划到底。

### 6. 什么时候用 Workflow 而不是开放式 Agent？

业务步骤固定、出错代价大、要合规审计时——审批、支付、改生产配置。LLM 放在意图识别、摘要、填字段这些软节点。

### 7. 如何防止无限调工具？

硬上限 + 相同参数重复调用熔断 + 编排层判断状态有无进展 + 失败后要求 `finish` 并说明原因，而不是无限 retry。

### 8. 工具调用失败怎么办？

分类型回传：参数错可让模型改一次；权限错直接停；5xx 有限重试；业务错给用户可读原因。别把堆栈丢给用户。

### 9. 高风险操作如何设计确认？

Preview 工具生成「将退款 300 元给订单 xxx」摘要，用户点确认后，**服务端**用另一 API 提交，带幂等键。确认内容要具体，不能只有「是否继续」。

### 10. Multi-Agent 一定更好吗？

不一定。沟通开销和一致性问题很现实。我倾向先 Workflow + 单 Agent，复杂审核再拆审查者角色。

### 11. Agent 记忆怎么设计？

分层：会话进度、短期事实、长期偏好。长期写入要门槛和删除入口，别把模型猜测永久化。

### 12. 如何评估能否上线？

任务集回放 + 工具/参数准确率 + 安全红队（注入、越权参数）+ 成本和 P95 延迟。没有工具链指标，只看最终文案会漏大问题。

### 13. 怎么降 Agent 成本？

减少无效循环、路由小模型、压缩 observation、缓存只读工具结果、能代码化的步骤别交给 LLM。

### 14. 设计「自动处理退款咨询」Agent 你怎么答？

我会选 Workflow：意图识别 → 查订单规则 → 满足条件生成说明，**真正退款 API 必须确认**；LLM 负责解释和异常摘要，规则和金额由确定性服务算。这样面试官能听到你懂风险分级。

### 15. 可观测性记什么？

request_id、prompt/model 版本、每步 tool_name/args/result/error、token、耗时、是否人工确认、最终输出和用户反馈。写操作还要操作者身份和幂等键，方便审计回放。
