# AI Agent 工具调用与 Function Calling 设计

## 核心概念

### 1. 工具调用在 Agent 中的位置

**工具调用（Tool Calling）** 是让大模型在对话中输出「调用哪个工具、传什么参数」，由**应用层**完成真实执行并把结果回灌给模型，从而继续推理或给出最终答案的机制。它是 Agent 区别于普通 Chatbot 的核心能力之一：模型负责**决策**，应用负责**执行与治理**。

典型闭环：

```text
用户目标
  → 模型根据 tools 定义生成 tool_calls（名称 + JSON 参数）
  → 应用层：解析 → Schema 校验 → 权限/租户绑定 → 限流/预算检查
  → 执行工具（HTTP、DB、搜索、业务 API、代码沙箱等）
  → 结构化结果写入 role=tool 消息
  → 模型基于结果：继续调用 / 改参重试 / 生成最终自然语言答复
```

面试时要能一句话说清价值：**把「不可控的自然语言」变成「可解析、可校验、可审计的结构化契约」**，从而让权限、幂等、重试、观测在工程上可落地。

### 2. Function Calling 与 Tool Calling

业界常混用，面试可这样区分：

| 概念 | 侧重 |
| --- | --- |
| **Function Calling** | 厂商 API 能力：模型输出 `function.name` + `arguments` JSON |
| **Tool Calling** | 更广：含 MCP 工具、工作流节点、检索器、Human-in-the-loop 确认门 |

OpenAI、Anthropic、Google Gemini 均提供 `tools` / `tool_use` 类接口；实现差异在消息格式、是否支持并行调用、是否支持 streaming tool arguments，但**设计原则一致**：契约在应用侧，执行权不在模型。

### 3. 三条铁律（必背）

| 原则 | 含义 | 反例 |
| --- | --- | --- |
| **模型只决策，不执行** | SQL、转账、发邮件由应用层跑 | 把 API Key 交给模型直连生产库 |
| **参数不可信** | 所有字段经 Schema + 业务规则校验 | `user_id` 由模型随便填就查库 |
| **工具即契约** | name、description、parameters、权限、错误协议一体设计 | 只有 OpenAPI，没有「何时不该调用」的说明 |

### 4. 多轮消息中的标准形态（OpenAI 风格）

```python
messages = [
    {"role": "system", "content": "你是客服助手。仅使用提供的工具查询订单，不得编造物流信息。"},
    {"role": "user", "content": "查订单 ORD-10086 的物流"},
    {
        "role": "assistant",
        "content": None,  # 仅有 tool_calls 时 content 常为 null
        "tool_calls": [{
            "id": "call_abc123",
            "type": "function",
            "function": {
                "name": "get_order_logistics",
                "arguments": '{"order_id": "ORD-10086"}'
            }
        }]
    },
    {
        "role": "tool",
        "tool_call_id": "call_abc123",
        # 中文注释：回灌结构化 JSON，避免整页 HTML
        "content": '{"status": "shipped", "carrier": "SF", "tracking_no": "SF123", "eta": "2026-05-24"}'
    },
    {
        "role": "assistant",
        "content": "您的订单 ORD-10086 已由顺丰发出，预计 5 月 24 日送达。"
    },
]
```

**常见断裂原因（排障考点）：**

- 漏传 `tool_call_id`，下一轮模型无法对齐哪次调用
- 未把带 `tool_calls` 的 assistant 消息原样 append，导致历史不完整
- 工具返回超长原文（整页 PDF、万行日志）直接回灌，挤爆上下文且引入 Injection
- `arguments` 是字符串而非对象——需在应用层 `json.loads`，失败要有明确错误协议

### 5. 与 MCP、Workflow 的关系

- **MCP（Model Context Protocol）**：标准化「客户端发现工具、调用工具」的协议层；工具定义仍要满足「契约 + 应用层执行」原则。
- **Workflow**：步骤固定、分支明确的流程用编排引擎更稳；工具调用适合**语义判断强、路径不固定**的节点。生产常见组合：**主流程 Workflow + 局部 LLM 选工具**。

---

## 核心知识点

### 1. 工具定义：四要素与反模式

完整工具定义 = **名称 + 描述 + 参数 Schema + 执行语义（权限/副作用/错误）**。

```json
{
  "type": "function",
  "function": {
    "name": "search_orders",
    "description": "按租户内用户 ID 查询近期订单列表，只读。用于「我的订单」「最近买了什么」类问题。用户未提供可解析的 user_id 时不要调用；退款、改地址请用其他工具。",
    "parameters": {
      "type": "object",
      "properties": {
        "user_id": {
          "type": "string",
          "description": "必须与当前登录用户一致，由服务端校验"
        },
        "days": {
          "type": "integer",
          "minimum": 1,
          "maximum": 90,
          "default": 30,
          "description": "查询最近 N 天，默认 30"
        },
      },
      "required": ["user_id"],
      "additionalProperties": false
    }
  }
}
```

**命名与粒度：**

- `name`：动词 + 宾语，如 `get_order`、`list_refund_policies`；避免 `handle_request` 这种万能工具
- **单一职责**：一个工具做一件事；「查 + 改 + 删」拆成多个，靠 description 约束边界
- **description 比参数名更重要**：写清「何时调用 / 何时不调用 / 缺参怎么办」，能显著降低选错工具率
- **Schema 严格化**：`enum`、`minimum/maximum`、`pattern`、`additionalProperties: false`；OpenAI 部分模型支持 **strict JSON schema**，减少幻觉字段

**反模式：**

| 反模式 | 后果 |
| --- | --- |
| 注册 50+ 工具不分类 | 选错、延迟高、成本高 |
| description 只写「查询订单」 | 与 `get_order_detail` 混淆 |
| 返回不可解析的长文本 | 上下文浪费、Injection 面扩大 |
| 写操作工具无幂等、无确认 | 重复扣款、误删数据 |

**动态工具子集（生产必备）：**

```python
# 中文注释：按意图只暴露 5～15 个相关工具，降低选错率
INTENT_TOOL_MAP = {
    "order_query": ["get_order", "list_orders", "get_logistics"],
    "refund_consult": ["get_order", "get_refund_policy"],  # 不暴露 submit_refund
    "chitchat": [],
}

def select_tools(user_message: str, session) -> list[dict]:
    intent = classify_intent(user_message)  # 小模型或规则
    names = INTENT_TOOL_MAP.get(intent, INTENT_TOOL_MAP["order_query"])
    return [registry.get_schema(n) for n in names]
```

### 2. 执行层：标准链路与横切能力

```python
MAX_TOOL_STEPS = 8
MAX_RETRY_PER_TOOL = 2

async def run_tool_loop(client, messages, tools, registry, ctx):
    """应用层工具循环：解析、校验、执行、回灌"""
    for step in range(MAX_TOOL_STEPS):
        resp = await client.chat.completions.create(
            model=ctx.model,
            messages=messages,
            tools=tools,
            tool_choice="auto",
            parallel_tool_calls=True,  # 读操作可并行，写操作在 registry 内串行
        )
        msg = resp.choices[0].message
        if not msg.tool_calls:
            return msg.content  # 最终自然语言答案

        messages.append(msg.model_dump(exclude_none=True))
        for tc in msg.tool_calls:
            try:
                args = json.loads(tc.function.arguments or "{}")
            except json.JSONDecodeError as e:
                result = {"error": "validation_error", "message": str(e), "retryable": True}
            else:
                if err := registry.validate(tc.function.name, args, ctx):
                    result = {"error": "validation_error", "message": err, "retryable": True}
                else:
                    result = await registry.execute(tc.function.name, args, ctx)
            messages.append({
                "role": "tool",
                "tool_call_id": tc.id,
                "content": json.dumps(truncate_for_context(result), ensure_ascii=False),
            })
    raise ToolBudgetExceeded(f"超过最大工具步数 {MAX_TOOL_STEPS}")
```

**横切能力清单（面试展开用）：**

| 能力 | 要点 |
| --- | --- |
| **身份绑定** | `user_id`、`tenant_id` 从 session 注入，禁止信任模型参数中的身份字段 |
| **权限** | RBAC/ABAC；工具元数据标 `read_only` / `write` / `admin` |
| **幂等** | 写操作带 `idempotency_key`；重复 tool_call 不重复副作用 |
| **超时与熔断** | 单工具超时、全局步数/token 预算 |
| **审计** | 记录 tool_name、args（脱敏）、结果摘要、trace_id |
| **限流** | 按用户/租户限制 QPS，防止 Agent 循环打爆下游 |

### 3. tool_choice 策略

| 取值 | 场景 |
| --- | --- |
| `auto` | 默认：模型决定是否调用 |
| `none` | 纯聊天、或敏感阶段禁止工具 |
| `required` | 必须先检索再答（如企业知识库问答的第一跳） |
| 指定 `function.name` | 强约束某一步必须调某工具（与 Workflow 节点等价） |

注意：`required` 不能替代权限校验；只是提高「一定会发起调用」的概率。

### 4. 并行 tool_calls 与依赖

模型可能在一条 assistant 消息里返回多个 `tool_calls`：

- **可并行**：只读、无共享写锁，如同时查订单 + 查物流 + 查优惠券
- **必须串行**：后一步依赖前一步结果；或写操作需严格顺序
- **实现**：`asyncio.gather` + 按 `tool_call_id` 顺序 append 多条 `role=tool` 消息（顺序一般不影响模型，但日志要对齐）

写操作并行风险：双重退款、库存超卖——**写工具默认串行 + 幂等键**。

### 5. 错误协议：让模型「可恢复」

工具失败不要只返回 `500` 或堆栈，应返回**结构化、可分类**的错误，便于模型改参或向用户解释：

```json
{
  "error": "not_found",
  "message": "订单 ORD-10086 不存在或不在当前租户下",
  "retryable": false,
  "suggestion": "请用户提供正确订单号或先调用 list_orders"
}
```

| error 类型 | 典型原因 | 模型/应用策略 |
| --- | --- | --- |
| `validation_error` | JSON 非法、缺必填、越界 | 允许有限次改参重试 |
| `permission_denied` | 跨租户、无 RBAC | 停止调用，提示用户 |
| `not_found` | 资源不存在 | 换条件或告知用户 |
| `conflict` | 状态不允许（已发货仍退款） | 说明业务规则 |
| `rate_limited` | 下游限流 | 退避重试或降级 |
| `confirmation_required` | 写操作未确认 | 走 HITL 流程 |
| `internal_error` | 依赖故障 | 有限重试后失败，勿泄露内部信息 |

**禁止回灌：** 完整 SQL、堆栈、内网 host、其他用户 PII。

### 6. 长返回与上下文压缩

工具返回往往比用户问题长得多，需**在回灌前**处理：

```python
def truncate_for_context(result: dict, max_tokens: int = 2000) -> dict:
    """中文注释：保留决策所需字段，列表 Top-N + total"""
    if "items" in result and isinstance(result["items"], list):
        total = len(result["items"])
        result = {
            **result,
            "items": result["items"][:20],
            "truncated": total > 20,
            "total": total,
        }
    return result
```

策略组合：

1. **服务端裁剪**：只返回 Top-N + `total`
2. **摘要工具**：`summarize_search_results` 二次调用（注意成本）
3. **二级加载**：先返回 ID 列表，再提供 `get_order_detail(order_id)` 按需拉取
4. **引用式 RAG**：检索片段带 `chunk_id`，详情另查

原则：**回灌内容服务于「下一步推理」**，不是把数据库 dump 给模型。

### 7. 高风险操作与安全

**预览 + 确认**：`preview_refund` 只读 → UI 确认 → 签发 `confirm_token` → `submit_refund` 校验 token。description 须写明无 token 禁止调用。

**Prompt Injection 防护**：untrusted 内容隔离标注；工具白名单；`user_id` 等从 session 绑定；高危工具 HITL；返回中隐藏指令过滤；下游 API 最小权限。

### 8. 与 Text-to-SQL、裸 HTTP 的取舍

| 方案 | 适用 | 风险 |
| --- | --- | --- |
| **预定义业务工具** | 生产主流 | 需维护工具集 |
| **Text-to-SQL** | 探索、内部分析 | SQL 注入、越权、幻觉表名 |
| **通用 HTTP 工具** | 集成多且变化快 | SSRF、凭证泄露 |

若必须 Text-to-SQL：**只读从库 + 表白名单 + 行级权限 + EXPLAIN 校验 + 结果行数上限**；写 SQL 一律禁止或走审批。

### 9. 流式 arguments、观测与厂商差异

**Streaming tool calls**：arguments 分片到达时需缓冲拼接后再 `json.loads`；配合 strict schema 降低半截 JSON。UI 可展示「正在查询订单…」提升体感。

**观测指标**：工具选择准确率、参数准确率、任务成功率、平均步数、P95 延迟、token 成本、安全违规率（越权/未确认写操作）。

**测试分层**：单元测 validate/权限/幂等；契约测 Schema 边界；回放测 messages 组装；端到端用「问题 → 期望工具链」标注集回归。

**厂商速记**：OpenAI 用 `tools` + `parallel_tool_calls`；Anthropic 用 `tool_use`/`tool_result`；Gemini 用 function declarations。建议 **ToolRegistry** 抽象，业务只关心 name + args + result。

---

## 高频面试问题与标准答案

### 1. Function Calling 解决什么问题？和普通 Prompt 里写 API 文档有何不同？

**标准答案：**  
Function Calling 的核心不是「让模型会调 API」，而是把输出约束成**可解析的结构化调用**（工具名 + JSON 参数），由应用层执行。这样在工程上可以统一做 Schema 校验、权限、幂等、审计、重试和观测。只在 Prompt 里贴 OpenAPI，模型仍可能输出自然语言、漏字段、编造参数；且无法在调用前拦截。价值在于**契约化边界**——模型负责意图和参数提议，系统负责「能不能做、对谁做、做了什么」。

### 2. 为什么不能让模型直接执行 SQL 或 HTTP？

**标准答案：**  
因为模型输出**不可信**：会有幻觉参数、Prompt Injection 诱导越权、跨租户访问。执行权必须在应用层：先做 JSON Schema 和业务校验，再把 `user_id` 从 session 绑定进去，而不是信任模型填的 user_id。还要做超时、限流、幂等和审计。直接让模型连生产库，等于把安全边界交给概率模型，生产不可接受。

### 3. 如何设计一个好的工具 Schema？description 和 parameters 哪个更重要？

**标准答案：**  
两者都重要，但 **description 往往更影响选错率**。要写清：这个工具做什么、不做什么、缺参数时该不该调用、和相邻工具的区别。parameters 侧用强类型：`enum` 限制状态、`minimum/maximum` 限制范围、`additionalProperties: false` 防止多余字段。工具粒度遵循单一职责，命名用 `get_order` 这种动词宾语结构。返回尽量是稳定 JSON，字段名固定，方便模型多轮推理。

### 4. 模型返回的 arguments 不是合法 JSON 怎么办？

**标准答案：**  
应用层 `json.loads` 失败时，不要静默跳过或随便填默认值，应构造 `validation_error` 回灌给模型，并限制**每工具重试次数**，防止死循环。预防手段包括：strict JSON schema、减少一次暴露的工具数量、复杂参数拆成多个简单工具、降低温度。若频繁失败，要查是不是 Schema 过复杂或 description 误导模型填了非 JSON 内容。

### 5. 为什么不要把公司所有 API 都注册成工具？

**标准答案：**  
工具越多，**选错工具、填错参数、成本和延迟**都会上升，攻击面也变大。生产做法是：按意图做动态子集，单次对话暴露大概 5～15 个相关工具；高危操作拆成 preview + confirm + submit，甚至不暴露给模型。能 Workflow 固定的步骤就不要赌模型每次选对工具。

### 6. 工具返回内容很长怎么处理？

**标准答案：**  
不能在回灌前把几万 token 的原文塞回上下文。常见做法：服务端只返回 Top-N 加 total、对列表做摘要、或者先返回 ID 再提供按需详情工具。日志里可以存全量，给模型的要是**服务于下一步决策**的精简结构。否则既浪费成本，又增加 Injection 和「迷失在中间上下文」的风险。

### 7. 并行 tool_calls 要注意什么？

**标准答案：**  
读操作、无依赖时可以 `asyncio.gather` 并行，降低延迟。写操作要默认串行，并加幂等键，避免重复提交。并行后每个结果必须对应正确的 `tool_call_id` 写回 messages。还要设全局步数上限，防止模型一轮发起过多调用把下游打挂。

### 8. 退款/删数据这类操作怎么设计？

**标准答案：**  
走**两阶段**：preview 工具只读，返回影响面；用户在前端确认后，服务端签发短期 `confirm_token`；submit 工具校验 token 与 session 一致才执行。description 里要写死「无 confirm_token 禁止调用」。确认权在人，不在模型。这和 Agent 可靠性、HITL 是同一套治理思路。

### 9. 如何防工具层的 Prompt Injection？

**标准答案：**  
外部内容不可信。手段包括：系统提示与 RAG/网页内容隔离标注；工具白名单；鉴权参数从 session 硬编码而不是模型参数；高危工具不注册或强制人工确认；工具返回里若出现「请调用某工具」要过滤和告警。核心思想是：**即使模型被诱导发了 tool_call，执行层仍要拦得住**。

### 10. 和 Text-to-SQL 方案怎么选？

**标准答案：**  
企业生产更推荐**预定义业务工具**，把 SQL 藏在工具实现里，模型只填业务参数。Text-to-SQL 适合内部分析、探索，若要用必须只读从库、表白名单、行级权限、结果行数限制，禁止写 SQL。Open-ended SQL 的越权和幻觉风险远高于结构化工具。

### 11. tool_choice 有哪些取值？什么时候用 required？

**标准答案：**  
`auto` 让模型自己决定；`none` 禁止工具；`required` 强制本轮必须发起工具调用，适合「必须先检索再回答」的 RAG 第一跳；还可以指定具体 function 名称。required 只是提高调用概率，不能替代权限校验。稳定流程更推荐 Workflow 固定调用顺序，而不是依赖模型自觉。

### 12. 如何测试和评估工具调用质量？

**标准答案：**  
分三层：单元测 validate、权限、幂等和错误分类；契约测每个工具 Schema 的边界样例；回放测多轮 messages 里 assistant/tool 消息是否完整。线上评工具选择准确率、参数准确率、任务成功率、平均步数和成本。Badcase 要保留完整 trace（脱敏），区分是选错工具、参数错还是下游业务错误。

---

## 面试回答加分点

1. **设计题顺序**：边界 → 工具列表 → Schema/description → 执行层 session 绑定与幂等 → 压缩回灌 → Injection → 评测指标。
2. **parallel_tool_calls**：读并行、写串行 + 幂等。
3. **Workflow 对比**：固定链路编排，LLM 只做意图与参数判断。
4. **ToolRegistry + strict schema**：屏蔽厂商差异，处理 streaming 半截 JSON。
5. **一分钟口述**：契约化 → 应用层鉴权执行 → 动态工具子集 → 错误可恢复 → preview+confirm → 评工具选择与参数准确率。
6. **延伸阅读**：同目录规划/多 Agent；`上下文工程`；MCP 文档。
