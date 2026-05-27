# LLM 结构化输出、解析与 Guardrails

## 主题选择记录

- **本次序号**：第 28 篇。
- **README 位置**：追加到目录表第 27 篇之后，归入「生产化与 LLMOps」卷。
- **分类目录**：`docs/知识库/提示词工程/`。
- **选择原因**：现有第 1 篇覆盖 Prompt 工程与结构化输出总览，第 3 篇覆盖 Function Calling，第 17 篇覆盖测试与 Mock，但「模型输出如何稳定落库、驱动下游流程、失败后如何解析与修复」还没有单独展开。面试中它常被追问为：JSON 总失败怎么办、Schema 怎么设计、Guardrails 是不是只靠 Prompt、如何防止脏数据进入业务系统。
- **避免重复说明**：本文不重复 Prompt 模板设计、Agent 工具调用闭环或测试体系总览，而聚焦 **输出契约、解析策略、校验修复、拒答与降级、上线观测** 这一条生产链路。

---

## 核心概念

LLM 结构化输出不是「让模型尽量按 JSON 回答」，而是把模型输出纳入工程契约：**模型负责生成候选结果，应用层负责解析、校验、修复、拒绝与审计**。面试时要强调：只要输出会进入数据库、触发工作流、调用工具或影响用户决策，就不能把自然语言结果直接信任。

典型链路如下：

```text
用户输入
  -> Prompt / JSON Schema / Tool Schema 约束
  -> LLM 生成
  -> 流式或完整输出解析
  -> 语法校验
  -> Schema 校验
  -> 业务规则校验
  -> 有限重试 / 修复 / 澄清 / 拒答
  -> 落库或驱动下游系统
  -> 观测、审计、坏 Case 回流
```

几个容易混淆的概念：

| 概念 | 解决什么问题 | 面试中的关键表达 |
| --- | --- | --- |
| JSON Mode | 降低非 JSON 文本概率 | 只能保证更像 JSON，不等于业务正确 |
| Structured Output | 用 Schema 约束字段、类型、枚举 | 输出是契约，不是普通文案 |
| Function Calling | 让模型产出工具名和参数 | 执行权仍在应用层 |
| Output Parser | 将文本转成对象 | 解析失败必须可分类、可回灌 |
| Guardrails | 规则、Schema、策略、模型评审组合 | 不是单一框架，也不是只写提示词 |

一个合格的结构化输出设计，至少回答三件事：

1. **要什么字段**：字段是否稳定、是否可为空、是否有枚举范围。
2. **如何验证**：语法、类型、业务规则、权限、证据来源是否可校验。
3. **失败怎么办**：重试、修复、澄清、降级还是拒绝，不能默认静默落库。

---

## 核心知识点

### 1. 结构化输出的本质是契约边界

自然语言适合展示给人，结构化结果适合交给系统。只要下游依赖字段，就应该把字段定义成契约，而不是在回答里约定「请按如下格式输出」后直接字符串截取。

反模式：

```text
请输出：姓名: xxx, 金额: xxx, 是否通过: xxx
```

问题在于字段名可能变化、金额可能带单位、是否通过可能写成「可以」「通过」「OK」。更稳的方式是定义严格结构：

```json
{
  "customer_name": "张三",
  "amount_cny": 12800.5,
  "approved": true,
  "risk_level": "medium",
  "evidence": ["合同第 3 页写明金额为 12800.5 元"]
}
```

面试回答重点：**模型输出只是候选对象，应用层必须把它当成不可信输入处理**。

### 2. Schema 设计要少、稳、可验证

Schema 不是越复杂越好。字段越多、嵌套越深，模型越容易漏填、编造或类型漂移。

```json
{
  "type": "object",
  "additionalProperties": false,
  "required": ["intent", "confidence", "slots"],
  "properties": {
    "intent": {
      "type": "string",
      "enum": ["query_order", "refund_request", "unknown"]
    },
    "confidence": {
      "type": "number",
      "minimum": 0,
      "maximum": 1
    },
    "slots": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "order_id": {
          "type": ["string", "null"],
          "pattern": "^ORD-[0-9]{8}$"
        }
      }
    }
  }
}
```

设计原则：

- **枚举优先**：意图、状态、风险等级尽量用 enum。
- **禁止额外字段**：`additionalProperties: false`，避免幻觉字段进入系统。
- **可空要显式**：不知道就返回 `null`，不要让模型猜。
- **字段语义稳定**：`amount_cny` 比 `amount` 更清楚。
- **复杂任务拆步**：先分类，再抽取；先召回证据，再生成结构化结果。

### 3. 解析失败要分类处理

结构化输出失败不是一个笼统的「模型不稳定」问题，至少要分层：

| 失败类型 | 例子 | 处理方式 |
| --- | --- | --- |
| 语法错误 | JSON 少括号、尾逗号 | 低成本修复或重试 |
| Schema 错误 | 少字段、类型错、枚举越界 | 回灌校验错误，要求按 Schema 修正 |
| 业务规则错误 | 金额为负、订单号不属于当前用户 | 终止或澄清，不能让模型自行绕过 |
| 证据不足 | 抽取字段无来源 | 返回 null / 低置信 / 人工复核 |
| 安全违规 | 输出敏感字段、越权结果 | 拒绝、脱敏、审计 |

示例：

```python
from pydantic import BaseModel, Field, ValidationError
from typing import Literal


class IntentResult(BaseModel):
    intent: Literal["query_order", "refund_request", "unknown"]
    confidence: float = Field(ge=0, le=1)
    order_id: str | None = None


def parse_intent(raw: str) -> IntentResult | dict:
    try:
        return IntentResult.model_validate_json(raw)
    except ValidationError as exc:
        # 中文注释：把可解释的校验错误回灌给模型或记录到坏 Case
        return {
            "error_type": "schema_validation_error",
            "details": exc.errors(),
            "retryable": True,
        }
```

面试中可以补一句：**解析失败不要吞掉，也不要随便填默认值；默认值会把模型错误变成业务脏数据**。

### 4. 重试不是无限循环，要有预算和错误回灌

重试的关键不是「再问一次」，而是带着明确错误信息修复。

```python
async def structured_generate(llm, messages, schema, max_retries: int = 2):
    last_error = None
    for attempt in range(max_retries + 1):
        response = await llm.chat(
            messages=messages + build_repair_message(last_error),
            response_schema=schema,
            temperature=0,
        )
        parsed = validate_response(response.content, schema)
        if parsed.ok:
            return parsed.value

        # 中文注释：只回灌必要错误，避免把完整内部规则暴露给模型
        last_error = {
            "error_type": parsed.error_type,
            "field": parsed.field,
            "message": parsed.message,
        }

    return {
        "status": "failed",
        "reason": "structured_output_validation_failed",
        "last_error": last_error,
    }
```

要点：

- 限制每次请求的重试次数。
- 只回灌必要校验信息，避免泄露内部策略。
- 区分可重试与不可重试错误。
- 高风险场景失败后进入人工复核或拒答，而不是继续生成。

### 5. Guardrails 是多层防线，不是一个 Prompt

常见误区是把 Guardrails 理解成「系统提示里写不要违规」。更可靠的分层如下：

```text
输入侧：长度、类型、注入检测、权限上下文
生成侧：低温度、Schema、工具白名单、少样例约束
解析侧：JSON parser、Schema validator、类型检查
业务侧：权限、金额、状态机、幂等、租户隔离
安全侧：敏感信息检测、脱敏、拒答策略、审计
上线侧：指标监控、坏 Case 回流、灰度与回滚
```

面试答法：**Prompt 是建议，Schema 是格式约束，业务规则才是强约束；强约束必须在服务端执行**。

### 6. 结构化输出与 Function Calling 的区别

- 结构化输出：目标是得到可解析对象，例如分类、抽取、评分、审批建议。
- Function Calling：目标是让模型提出工具调用意图和参数，由应用执行工具。

二者经常组合：先用结构化输出识别意图，再决定是否暴露工具；或让 Function Calling 产出参数后，再用业务 Schema 校验。

关键边界：

```python
def execute_refund(args: dict, session: dict) -> dict:
    order_id = args["order_id"]
    amount = args["amount"]

    # 中文注释：user_id 从登录态绑定，不能相信模型参数里的用户身份
    if not order_belongs_to_user(order_id, session["user_id"]):
        return {"error": "permission_denied"}

    if amount <= 0 or amount > refundable_amount(order_id):
        return {"error": "business_rule_violation"}

    return create_refund(order_id=order_id, amount=amount)
```

### 7. 流式输出场景更要谨慎

流式适合提升体验，但结构化 JSON 在流式过程中经常是半截状态。不要在未完成前驱动下游系统。

处理方式：

- 展示层可以流式显示自然语言解释。
- 机器消费字段应等 `done` 后完整解析。
- 如需边流边解析，必须使用增量 parser，并且只把通过校验的完整事件交给下游。
- 中途断连要取消上游生成，避免后台继续消耗 token。

面试高频点：**半截 JSON 不能落库，半截工具参数不能执行**。

### 8. RAG 场景要把证据绑定到字段

在知识库、合同、票据、医疗等场景，仅输出字段值不够，还要输出来源证据。

```json
{
  "invoice_code": "044002100111",
  "total_amount": 5600.0,
  "currency": "CNY",
  "evidence": [
    {
      "field": "total_amount",
      "doc_id": "invoice_2026_001",
      "page": 1,
      "quote": "价税合计（大写）伍仟陆佰圆整 ￥5600.00"
    }
  ]
}
```

字段没有证据时，应返回 `null` 或进入复核。不要让模型凭常识补全合同金额、身份证号、日期等高风险字段。

### 9. 评估指标不能只看 JSON 合法率

结构化输出的评估至少包括：

| 指标 | 说明 |
| --- | --- |
| JSON 合法率 | 是否能被 parser 解析 |
| Schema 通过率 | 类型、必填、枚举是否满足 |
| 字段准确率 | 每个字段是否抽取正确 |
| 业务规则通过率 | 是否满足权限、金额、状态机等约束 |
| 证据一致率 | 字段值是否能被引用证据支持 |
| 修复成功率 | 第一次失败后能否在有限重试内修复 |
| 人工复核命中率 | 低置信样本是否真的值得复核 |

上线后还要监控字段缺失率、重试率、拒答率、人工复核率、下游报错率和坏 Case 类型分布。

### 10. 面试中的系统设计模板

遇到「设计一个用 LLM 自动抽取表单/审批/工单分类系统」时，可以按这条线回答：

1. 定义输出 Schema：字段、类型、枚举、可空、证据。
2. Prompt 或模型 API 开启结构化输出能力。
3. 服务端做 parser + Schema 校验。
4. 加业务校验：权限、状态机、金额边界、租户隔离。
5. 失败分流：可重试、需澄清、需人审、拒绝。
6. 结果落库前记录原始输出、解析结果、模型版本、Prompt 版本。
7. 建评测集和线上指标，坏 Case 回流迭代。

---

## 高频面试问题与标准答案

**Q1：结构化输出和普通 Prompt 要求 JSON 有什么区别？**
普通 Prompt 只是文本约定，模型仍可能漏字段、输出解释文字或编造格式。结构化输出强调契约化：用 JSON Schema、Pydantic 或模型 API 的 response format 约束字段，并在服务端做解析、Schema 校验和业务校验。我的理解是：Prompt 让模型「知道该怎么答」，Schema 和应用层校验决定「结果能不能被系统接收」。

**Q2：LLM 输出的 JSON 经常解析失败，你会怎么处理？**
我会先把失败分类：是语法错误、字段缺失、类型错误、枚举越界，还是业务规则不满足。语法和 Schema 问题可以把精简后的校验错误回灌给模型，限制 1 到 2 次重试；业务规则错误不能靠模型自行修，要直接拒绝、澄清或进入人工复核。同时会降低温度、简化 Schema、减少嵌套，并监控解析失败率和具体字段分布。

**Q3：为什么不能解析失败后给默认值？**
默认值会把模型错误伪装成正常业务数据，尤其是金额、时间、审批结果、权限字段这类高风险字段。更好的做法是显式返回 `null`、低置信或校验错误，让下游知道这个字段不可靠。只有业务上真正有稳定默认语义的字段，才应该由服务端填默认值，而不是让模型错误触发默认逻辑。

**Q4：Schema 应该怎么设计才稳定？**
我会让 Schema 尽量小而明确：字段名表达业务语义，枚举值固定，数字有范围，字符串有 pattern，高风险字段可空并要求证据，禁止额外字段。复杂任务会拆成多步，比如先分类意图，再抽取槽位，而不是一次要求模型输出一个很深的对象。这样既降低模型出错率，也方便评估每一步。

**Q5：Guardrails 是不是在系统提示词里写安全规则？**
不是。系统提示词只是软约束，真正可靠的 Guardrails 是多层防线：输入过滤、结构化输出、Schema 校验、权限校验、业务规则、敏感信息脱敏、人工复核、日志审计和线上监控。面试里我会强调：凡是强约束都要在服务端执行，不能只相信模型会遵守提示词。

**Q6：结构化输出和 Function Calling 怎么取舍？**
如果只是分类、抽取、评分、生成一个可落库对象，用结构化输出就够了；如果模型需要建议调用外部系统，比如查订单、退款、创建工单，就用 Function Calling 或 Tool Calling。但 Function Calling 的参数也属于模型输出，仍要做 Schema、权限、业务规则和幂等校验。两者不是替代关系，很多生产系统会组合使用。

**Q7：RAG 抽取结果如何保证可信？**
我会要求每个关键字段带证据，比如 doc_id、页码、原文 quote 或 span。服务端校验字段值是否能被证据支持，证据不足就返回 null 或进入人工复核，而不是让模型凭上下文补全。上线评估不能只看 JSON 合法率，还要看字段准确率、证据一致率和人工复核命中率。

**Q8：流式输出 JSON 时，下游能不能边收到边处理？**
要看处理对象。如果是展示给用户的自然语言，可以边流边展示；如果是机器消费的 JSON 字段，通常要等完整 `done` 后再解析和校验。半截 JSON 不能落库，也不能触发工具调用。若业务确实需要边流边处理，要设计结构化事件协议和增量 parser，只提交已经完整且通过校验的事件。

**Q9：结构化输出上线后看哪些指标？**
我会看 JSON 合法率、Schema 通过率、字段准确率、业务规则失败率、重试率、修复成功率、拒答率、人工复核率、下游报错率和坏 Case 分布。只看 JSON 合法率不够，因为合法 JSON 也可能字段错、证据不支持或违反业务规则。

**Q10：如果面试官让你设计一个合同关键信息抽取系统，你怎么答？**
我会先定义合同字段 Schema，比如主体、金额、日期、付款条件、违约条款，并要求每个字段带页码和原文证据。链路上先做文档解析和切分，再检索相关片段，模型按 Schema 抽取，服务端做类型、金额、日期和证据校验。低置信或证据冲突的字段进入人工复核，结果落库时记录模型版本、Prompt 版本和证据，后续用人工修正样本回流评测集。
