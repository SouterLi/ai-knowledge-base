# LLM 应用测试与 Mock 策略

## 核心概念

### 1. 测试金字塔（LLM 版）

```text
端到端回放（可选真实模型）     ← 少、贵、看质量
集成测试（Mock 模型/工具）     ← 验证编排
契约测试（Prompt/Schema/Tool） ← 稳定接口
单元测试（解析/权限/路由）     ← 多、快、确定
```

**核心**：把不可控的模型输出拆成可控工程边界——工程逻辑确定性测，模型质量阶段性评。

### 2. Mock / Stub / Fake / Replay

| 类型 | 作用 | 例子 |
| --- | --- | --- |
| Stub | 固定返回 | 固定 JSON 回答 |
| Mock | 验证调用行为 | 断言 temperature、tools |
| Fake | 可运行轻量实现 | 内存向量库 |
| Replay | 录制回放 | 线上响应录制 |

Mock 目的：**隔离变量**，不是证明模型永远正确。

### 3. 黄金样例

人工确认的关键场景输入 + 期望（事实点、禁止项、格式、工具路径），**不要求逐字匹配**。

### 4. 契约

凡下游依赖的字段/格式/行为都是契约：Prompt 变量完整性、JSON Schema、Tool 参数、错误码语义、检索返回字段。

---

## 核心知识点

### 1. 确定性单元测试范围

变量渲染、JSON 解析/修复、权限、路由、重试退避、缓存 key、工具参数校验。

```python
def validate_search_args(args: dict) -> dict:
  if not args.get("query"):
    raise ValueError("query 不能为空")
  limit = args.get("limit", 5)
  if not 1 <= limit <= 20:
    raise ValueError("limit 必须在 1 到 20")
  return {"query": args["query"], "limit": limit}
```

### 2. Mock 模型客户端

```python
class LLMClient:
  def chat(self, messages, tools=None, temperature=0):
    raise NotImplementedError

class FakeLLMClient(LLMClient):
  def __init__(self, response):
    self.response = response
    self.calls = []

  def chat(self, messages, tools=None, temperature=0):
    self.calls.append({"messages": messages, "tools": tools, "temperature": temperature})
    return self.response
```

测的是**请求构造是否正确**，不是模型能力。

### 3. Prompt 测试

- 模板：变量完整、无注入风险、含关键约束。
- 回归：固定上下文下断言事实点/禁止项/长度。
- 对抗：注入、越权、恶意格式是否拒绝。

```python
def assert_answer_quality(answer: str):
  assert "退款申请" in answer
  assert "数据库管理员" not in answer
  assert len(answer) <= 300
```

### 4. Tool Calling 测试

断言：选对工具、参数合法、失败不编造、危险操作需确认。

```python
def test_agent_calls_shipping_tool():
  tool = FakeTool("get_order_shipping_status", {"status": "运输中"})
  agent = Agent(tools=[tool], llm=FakeLLMClient({...}))
  agent.run("查订单 A1001 物流")
  assert tool.calls == [{"order_id": "A1001"}]
```

### 5. 非确定性处理

- temperature=0、固定模型版本、固定检索与工具结果。
- 断言事实/字段/工具路径，非完整字符串。
- CI：PR 用 Mock；发布前跑真实模型评测。

### 6. 回放测试

录制：输入、上下文、检索结果、Prompt/模型版本、工具请求响应、输出、失败原因。覆盖高频、高风险、历史坏 Case。

### 7. CI 分层

| 阶段 | 内容 | 真实模型 |
| --- | --- | --- |
| PR | 单元+契约+Mock 集成 | 否 |
| 合并前 | 小规模黄金样例 | 可选 |
| 发布前 | 完整离线评测+安全 | 是 |

---

## 高频面试问题与标准答案

**Q1：输出不稳定怎么写自动化测试？**

分层：确定性逻辑单元测；模型 Mock/Replay；结构化 Schema 校验；自然语言用事实点/禁止项；关键场景真实模型黄金样例。不全文匹配。

**Q2：为什么不能所有测试都调真模型？**

成本高、慢、波动大、受限流/版本影响。真模型适合发布评测；工程逻辑应 Mock，失败可定位到代码而非模型随机性。

**Q3：如何合理 Mock LLM 客户端？**

封装 `chat(messages, tools, temperature)` 接口；注入 Fake，固定返回并记录调用参数，验证 Prompt 与参数传递。

**Q4：Prompt 修改如何防回归？**

维护黄金样例（高频/边界/安全/坏 Case）；固定检索与工具；检查事实、格式、引用、拒答；发布前小规模真模型评测。

**Q5：Tool Calling 怎么测？**

三层：选对工具、参数合法且有权限、最终结果正确使用工具返回；覆盖失败/超时/重复调用/未确认写操作。

**Q6：LLM-as-Judge 能用吗？**

可作开放式回答初筛，不能单独作高风险最终裁判；需固定评分标准与模型版本，并人工抽样校验一致性。

**Q7：如何测流式输出？**

测事件协议顺序（start→delta→done）、取消时上游停止、拼接等于最终文本；用 Fake Stream 逐块返回。

**Q8：线上坏 Case 怎么处理？**

补全上下文 → 归因（检索/Prompt/工具/权限/模型）→ 修复 → 加入回归集（必须包含/禁止断言）。

---

## 面试回答加分点

1. 开场：**分层、隔离、契约化、回归化** 四原则。
2. 强调 Mock **隔离变量**，不是替代质量评测。
3. CI 区分 PR（快）与发布前（真模型），体现成本意识。
4. Tool 危险操作提**人类确认**，与安全/权限测试联动。
5. 固定**模型版本与 Prompt 版本**，避免测试漂移。
6. 坏 Case **回流**是长期资产，不只修一次。
7. 一句话：**工程行为 Mock 验证，模型质量黄金样例把关**。
