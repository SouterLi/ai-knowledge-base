# Prompt 工程与结构化输出

## 核心概念

**Prompt 工程**不是「写一段好话术」，而是把业务目标、上下文、约束、输出格式和失败处理清晰地交给模型，使 Prompt 成为**可测试、可迭代、可观测**的工程资产。

**结构化输出**指模型按固定格式返回（JSON、枚举、表格、函数参数），用于信息抽取、分类、工单流转、审核、自动填表和 API 前置参数生成。与自然语言相比，结构化输出便于程序解析、校验和接入下游流程。

**高质量 Prompt 六要素：**

| 要素 | 说明 |
| --- | --- |
| 角色 | 模型承担什么职责 |
| 任务 | 单一明确目标，不混入多个模糊目标 |
| 上下文 | 必要材料、用户输入、业务规则 |
| 约束 | 不能编造、仅返回给定枚举等 |
| 输出格式 | 字段、类型、示例、缺省值 |
| 质量标准 | 何为好答案、何时拒答或返回空值 |

**核心分工：** 模型生成**候选内容**；系统负责**解析、校验、兜底**。Prompt 不能替代权限、金额阈值、合规红线——硬规则在代码或规则引擎。

```python
# 结构化输出链路
async def extract_intent(user_text: str) -> dict:
    raw = await llm.chat(
        system=PROMPT_V3,  # 版本化 Prompt
        user=user_text,
        response_format={"type": "json_object"},  # 或 function calling
    )
    data = json.loads(raw)
    validated = IntentSchema.model_validate(data)  # Pydantic 校验
    if validated.confidence < 0.6:
        return {"need_human": True, "draft": validated.model_dump()}
    return apply_business_rules(validated)  # 中文注释：业务规则在代码层
```

---

## 核心知识点

### 1. 先定义下游消费方式

给人看 → 表达清晰；给程序处理 → **格式稳定是系统契约**，不是最后修饰。

### 2. Schema 设计

```json
{
  "intent": "refund | change_address | unknown",
  "confidence": 0.0,
  "summary": "一句话概括用户诉求",
  "need_human": false
}
```

工程实践：JSON Schema / Pydantic / Zod 服务端校验；`additionalProperties: false`；关键字段用 `enum` 收敛开放文本。

### 3. 上下文与位置策略

- 只注入与任务相关的上下文；长材料先分段、摘要或 RAG
- **强约束靠近输出要求**（部分模型对尾部指令更敏感）
- 控制长度，避免关键规则被淹没

### 4. 解析、校验与重试

```python
def parse_with_retry(raw: str, schema, max_retry: int = 2) -> dict:
    for attempt in range(max_retry + 1):
        try:
            data = json.loads(raw)
            return schema.validate(data)
        except (json.JSONDecodeError, ValidationError) as e:
            if attempt == max_retry:
                raise
            # 中文注释：把具体错误反馈给模型，而非重复同一 Prompt
            raw = await llm.chat(user=f"上次错误: {e}，请严格按 Schema 重试")
    raise RuntimeError("unreachable")
```

重试策略：降低 `temperature`；简化任务；使用 JSON mode / strict schema；失败降级人工或默认枚举 `unknown`。

### 5. Few-shot 示例

不是越多越好。少量**覆盖典型边界**的示例优于大量重复示例。示例须与规则和 Schema 一致，避免冲突。

### 6. Prompt Injection 防护

- 用户输入与系统指令分离，明确边界标记（如 `<user_input>...</user_input>`）
- 声明 untrusted 区域中的指令无效
- 敏感操作权限在应用层校验
- 输出白名单检查，不能仅靠「忽略恶意指令」

### 7. 版本化与 CI

- Prompt 版本号、变更原因、适用模型、评估结果
- 日志：输入摘要、Prompt 版本、模型版本、解析错误、重试次数
- Prompt 测试纳入 CI，防回归

### 8. 与 Function Calling 的关系

结构化字段抽取可用 JSON mode；工具参数生成用 Function/Tool Calling。二者都需服务端 Schema 校验，不能只信模型输出。

---

## 高频面试问题与标准答案

**1. Prompt 工程本质是什么？**  
把不稳定的 NL 交互变成可验证的工程接口：任务清晰、格式稳定、服务端校验、失败重试、评估迭代、安全边界。

**2. 模型为何不按格式输出？**  
格式不明确、示例与规则冲突、任务过复杂、温度过高、未用 JSON mode。解决：简化任务、明确 Schema、降 temperature、服务端强制校验。

**3. Few-shot 越多越好吗？**  
否。占用上下文、引入噪声。高质量少量边界示例更有效。

**4. Prompt 能替代业务规则吗？**  
不能。语义判断用 Prompt；权限、金额、合规、状态流转用代码/规则引擎。模型输出是候选，最终决策需系统校验。

**5. 如何处理注入攻击？**  
分离指令与 untrusted 内容、边界标记、应用层权限、输出白名单；Agent 场景还需限制工具权限。

**6. 结构化输出完整链路？**  
调用模型 → 解析 JSON → Schema 校验 → 业务规则 → 失败重试（带错误反馈）或降级/人工。

**7. 如何评估 Prompt 效果？**  
样例集覆盖正常/边界/歧义/恶意/空输入；指标：准确率、格式通过率、拒答正确率、延迟、成本；A/B 不同 Prompt 版本。

**8. JSON mode 与 Prompt 里写「返回 JSON」区别？**  
JSON mode / response_format 在 API 层约束输出形态，通过率更高；仍须服务端校验，因模型可能漏字段或越 enum。

**9. 低置信度如何处理？**  
`need_human: true`、降级话术、转人工队列；不在低置信时自动执行高风险动作。

**10. 如何迭代 Prompt？**  
收集 bad case → 归因（任务/格式/上下文/模型）→ 小步修改单变量 → 回归评测集 → 版本发布与监控。

---

## 面试回答加分点

1. **强调「契约思维」**：输出 Schema 是下游 API 的一部分，与数据库表设计同级严肃。
2. **给出 Pydantic/重试代码片段**，体现工程落地而非只背概念。
3. **区分 Prompt 与 RAG**：RAG 补外部事实；Prompt 定义任务与格式；二者在 `build_messages` 中分层组装。
4. **提及 temperature**：抽取/分类用 0～0.3；创意写作用更高，但结构化节点仍偏低。
5. **安全题串联**：Prompt 注入 + 工具层校验 + 人工确认，三层防御。
6. **一句话收束**：模型负责生成候选，系统负责解析、校验和兜底；结构化输出的关键是格式稳定与可验证，而非「让模型听话」。
