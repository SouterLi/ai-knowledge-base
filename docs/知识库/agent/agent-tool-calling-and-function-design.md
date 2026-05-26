# AI Agent 工具调用与 Function Calling 设计

## 核心概念

**工具调用的本质**是把外部能力结构化、可校验、可审计地暴露给模型，而非「给模型更多 API」：

```text
用户目标 → 模型输出「调用意图+参数」→ 应用层校验 → 执行工具 → 结果回灌 → 模型继续或结束
```

**三条铁律（面试必答）：**

| 原则 | 说明 |
| --- | --- |
| 模型只决策，不执行 | SQL、HTTP、发邮件由应用层完成 |
| 参数不可信 | 必须经过 Schema + 业务规则校验 |
| 工具即契约 | 名称 + 描述 + 参数 Schema + 权限 + 错误协议 |

**Function Calling vs Tool Calling：** 实践中常混用。Function Calling 强调「NL → 结构化调用」；Tool Calling 范围更广（HTTP、MCP、数据库、工作流节点）。OpenAI/Anthropic/Gemini 均提供 `tools` API。

**上下文中的消息形态（OpenAI 风格）：**

```python
messages = [
    {"role": "user", "content": "查订单 ORD-10086 的物流"},
    {
        "role": "assistant",
        "tool_calls": [{
            "id": "call_abc",
            "type": "function",
            "function": {
                "name": "get_order_logistics",
                "arguments": '{"order_id": "ORD-10086"}'
            }
        }]
    },
    {
        "role": "tool",
        "tool_call_id": "call_abc",
        # 中文注释：结构化 JSON，勿塞几万字 HTML
        "content": '{"status": "shipped", "carrier": "SF", "eta": "2026-05-24"}'
    },
]
```

易错点：漏传 `tool_call_id`、工具原始长文本整段回灌、assistant 消息未保留 `tool_calls` 导致多轮断裂。

---

## 核心知识点

### 1. 工具定义四要素

```json
{
  "type": "function",
  "function": {
    "name": "search_orders",
    "description": "按用户ID查近期订单。仅用于查询，不用于退款。用户未提供 order_id 时不要调用。",
    "parameters": {
      "type": "object",
      "properties": {
        "user_id": {"type": "string", "description": "租户内用户唯一标识"},
        "days": {"type": "integer", "minimum": 1, "maximum": 90, "default": 30}
      },
      "required": ["user_id"],
      "additionalProperties": false
    }
  }
}
```

- **name**：动词+宾语，如 `get_order`
- **description**：写清何时调用/不调用，比参数名更能降选错率
- **parameters**：`enum`、`minimum/maximum`、`additionalProperties: false`
- **权限元数据**：应用层扩展 `read_only` / `write` / `requires_confirmation`

单次暴露建议 **5～15 个**；过多则按意图做**动态工具子集**。

### 2. 执行层标准链路

```python
async def run_tool_loop(client, messages, tools, registry, ctx):
    for step in range(MAX_STEPS):
        resp = await client.chat.completions.create(
            model="gpt-4o", messages=messages, tools=tools, tool_choice="auto"
        )
        msg = resp.choices[0].message
        if not msg.tool_calls:
            return msg.content
        messages.append(msg.model_dump())
        for tc in msg.tool_calls:
            try:
                args = json.loads(tc.function.arguments)
            except json.JSONDecodeError:
                args = {}
            err = registry.validate(tc.function.name, args, ctx)
            if err:
                result = {"error": "validation_error", "message": err}
            else:
                result = await registry.execute(tc.function.name, args, ctx)
            messages.append({
                "role": "tool",
                "tool_call_id": tc.id,
                "content": json.dumps(truncate_result(result), ensure_ascii=False),
            })
    raise RuntimeError("max tool steps exceeded")
```

### 3. tool_choice 策略

| 取值 | 用途 |
| --- | --- |
| `auto` | 模型决定是否调用 |
| `none` | 禁止工具 |
| `required` | 强制调用（如必须先检索） |
| 指定 function | 强制调用某工具 |

### 4. 并行 tool_calls

- **无依赖读操作**：`asyncio.gather` 并行
- **写操作**：串行 + 幂等键
- 并行后按 `tool_call_id` 组装 messages

### 5. 错误协议

```json
{"error": "permission_denied", "message": "无权访问 order_id=ORD-99999", "retryable": false}
```

| 类型 | 模型策略 |
| --- | --- |
| validation_error | 改参重试（限次数） |
| permission_denied | 停止或提示用户 |
| not_found | 换条件或告知用户 |
| rate_limited | 等待或降级 |
| internal_error | 有限重试后失败 |

勿返回堆栈、SQL、内部 host。

### 6. 动态工具子集

```python
INTENT_TOOLS = {
    "order": ["get_order", "list_orders", "get_logistics"],
    "refund": ["get_order", "get_refund_policy"],  # 中文注释：退款执行工具不暴露
    "chitchat": [],
}
```

### 7. 安全：工具层 Prompt Injection

外部网页/RAG 可能出现「调用 delete_all_users」。防护：系统指令与 untrusted 分离；工具白名单；`user_id` 绑定 session 硬校验；高风险工具不注册或需确认；返回内容过滤隐藏指令。

### 8. 与 Text-to-SQL 对比

生产推荐预定义工具 `query_orders(user_id, days)`；禁用模型直接生成可写 SQL；若 Text-to-SQL 则只读从库+表白名单+行级权限。

---

## 高频面试问题与标准答案

**1. Function Calling 解决什么？**  
把输出约束为可解析的结构化调用（工具名+JSON 参数），应用层执行，使错误处理、权限、审计可工程化。价值在**契约化边界**。

**2. 为何不能让模型直接执行？**  
幻觉参数、Injection、跨租户访问。应用层持有执行权：Schema 校验、身份绑定、权限、幂等、超时、审计。

**3. 如何设计好 Schema？**  
单一职责；description 写边界；强类型+`additionalProperties: false`；返回稳定 JSON；高风险标 `requires_confirmation`。

**4. arguments 非合法 JSON？**  
`json.loads` 失败→回灌 `validation_error` 允许重试；预防：strict schema、固定单工具、限制步数。勿静默执行。

**5. 为何不能把所有 API 都注册？**  
选错率、成本、权限面扩大。最小集合；高危改为预览+确认+提交；意图路由动态挂载。

**6. 工具返回很长？**  
Top-N+`total`、服务端摘要、二级按需加载工具、日志先摘要再回灌。原则：服务于下一步推理。

**7. 预览+确认两阶段？**  
`preview_refund` 只读返回金额与风险；`submit_refund` 校验 `confirm_token` 来自用户点击。**确认权在人**。

**8. 如何测试？**  
单元测 validate/权限/幂等；契约测 Schema 样例；回放测多轮消息组装；端到端评工具选择与参数准确率。

---

## 面试回答加分点

1. **设计题顺序**：边界（只读）→ 工具列表（2～3 个不重叠）→ Schema → 执行层 session 交叉校验 → 压缩回灌 → 防 Injection → 评测指标。
2. **提及 parallel_tool_calls** 与写操作串行的取舍。
3. **对比 Workflow**：稳定流程用编排固定检索顺序，而非 `tool_choice` 赌模型自觉。
4. **OpenAI**：新版统一 `tools`（`type: function`），注意 strict JSON schema、`parallel_tool_calls`。
5. **一分钟口述**：结构化契约 → 应用层解析鉴权执行 → description 降选错 → 白名单+参数绑定 → 长返回压缩 → 评工具选择与参数准确率。
6. **与相关主题分工**：本文=工具协议；规划可靠性见同目录规划文档；上下文见 context-engineering；Prompt 见提示词工程。
