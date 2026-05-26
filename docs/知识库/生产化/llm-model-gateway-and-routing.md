# LLM 模型网关与多模型路由

## 核心概念

### 1. 模型网关

模型网关是业务系统与模型供应商之间的**统一调用层**，职责包括：

- **协议统一**：屏蔽 OpenAI、Claude、Gemini、本地模型等接口差异。
- **治理集中**：鉴权、限流、重试、超时、熔断、降级、缓存、审计、成本统计。
- **配置入口**：模型别名、版本、供应商权重、灰度规则。

核心价值不是“代理转发”，而是**治理模型调用的复杂度**。

### 2. 模型路由

根据任务类型、质量要求、性能目标、成本约束和**运行时健康状态**选择模型。成熟方案包含：

- **静态路由**：配置驱动，按任务/租户/场景选模型。
- **动态路由**：根据错误率、P95 延迟、429 数量实时调整。

### 3. 模型别名

业务依赖稳定别名（如 `chat_fast`、`chat_reasoning`），不直接绑定具体模型名，便于升级、切换供应商和灰度实验。

### 4. 降级、熔断与兜底

| 机制 | 含义 | 注意 |
| --- | --- | --- |
| 降级 | 强模型失败切小模型、短答案或异步 | 医疗/金融等高风险场景禁止不安全降级 |
| 熔断 | 错误率超阈值后短期停止请求该 Provider | 需半开探测恢复 |
| 兜底 | 可解释失败、排队、人工介入 | 不能静默丢请求 |

---

## 核心知识点

### 1. 推荐架构

```text
业务服务 → 模型网关 → 路由器 → Provider Adapter → 供应商
         ↘ 校验 / 审计 / 观测
```

业务只传**模型别名**和任务参数，不绑定供应商。

### 2. 路由策略

| 策略 | 场景 | 要点 |
| --- | --- | --- |
| 规则路由 | 任务类型明确 | 可解释、易灰度 |
| 权重路由 | A/B、灰度 | 按用户/租户稳定分桶 |
| 成本优先 | 大量低风险请求 | 小模型先试，不达标再升级 |
| 健康路由 | 供应商波动 | 避开错误率高的 Provider |

每次路由记录**命中规则、实际模型、降级原因**，便于回放。

### 3. Provider Adapter

统一输入（messages、tools、response_format）和输出（content、usage、finish_reason、错误码），把不同 SSE chunk 转成统一流式事件。

```python
class ProviderAdapter:
  def complete(self, request: dict) -> dict:
    raise NotImplementedError

class OpenAIAdapter(ProviderAdapter):
  def complete(self, request: dict) -> dict:
    # 中文注释：转换字段、设置超时、解析 usage 与统一错误码
    return {
      "content": "...",
      "usage": {"input_tokens": 120, "output_tokens": 80},
      "provider": "openai",
    }
```

### 4. 超时、重试与幂等

- 仅对超时、429、部分 5xx 做**有限重试**（退避 + 上限）。
- 鉴权失败、参数错误、内容安全拒绝**不重试**。
- 流式已输出后重试会导致重复片段，需特殊处理。
- 写操作工具调用必须带**幂等键**。

```python
RETRYABLE = {"timeout", "rate_limited", "provider_5xx"}

def should_retry(code: str, attempt: int, max_attempts: int = 2) -> bool:
  # 中文注释：避免把业务错误放大成流量风暴
  return code in RETRYABLE and attempt < max_attempts
```

### 5. 熔断与健康检查

指标：错误率、P95/P99、429/5xx、流式中断率、探活结果。熔断后进入半开，少量探测验证恢复。

```python
def select_provider(candidates: list, health: dict) -> str:
  available = [p for p in candidates if health[p]["state"] != "open_circuit"]
  if not available:
    raise RuntimeError("no_available_provider")
  return min(available, key=lambda p: health[p]["p95_latency_ms"])
```

### 6. 灰度与可观测

流程：离线评测 → 影子流量 → 小流量稳定分桶 → 指标对比 → 配置回滚。

日志字段：request_id、tenant_id、model_alias、selected_model、route_rule、fallback_reason、token、cost、ttft；**不落完整 Prompt 和密钥**。

```python
MODEL_ROUTES = {
  "chat_fast": [
    {"provider": "a", "model": "small", "weight": 80},
    {"provider": "b", "model": "small", "weight": 20},
  ],
}

def route_model(alias: str, health: dict) -> dict:
  for c in MODEL_ROUTES[alias]:
    if health[c["provider"]]["state"] == "healthy":
      return c
  raise RuntimeError("no_healthy_model")
```

---

## 高频面试问题与标准答案

**Q1：为什么需要模型网关？**

小规模 Demo 可直接调 API；生产需要统一治理鉴权、限流、重试、熔断、降级、审计、成本与灰度。否则各业务重复封装，模型升级和故障切换不可控。

**Q2：如何设计多模型路由？**

定义模型别名 → 按任务/租户/成本/延迟/质量制定规则 → 结合健康状态动态避开故障供应商 → 记录路由决策与降级原因。

**Q3：供应商突然不可用怎么办？**

根据错误率/超时触发熔断，切备用 Provider 或模型；低风险可降级小模型，高风险返回可解释失败或转人工；恢复时半开探测再逐步放量。

**Q4：模型升级如何灰度？**

离线评测 → 影子流量（不影响用户）→ 按用户/租户稳定分桶小流量 → 对比解析失败率、投诉、成本、P95、安全拒答 → 配置层回滚，无需发版。

**Q5：网关日志记什么？有什么风险？**

记 request_id、租户、场景、别名、实际模型、Prompt 版本、token、成本、延迟、错误码、重试与降级原因。风险是泄露隐私和商业数据，需脱敏、采样与分级授权。

**Q6：如何兼顾成本与质量？**

按任务风险分层：低风险小模型优先，置信度不足再升级；高风险直接强模型 + 结构化校验。用评测集验证“便宜模型是否足够好”。

**Q7：重试、降级、熔断的关系？**

重试处理单次短暂失败；降级在能力或体验下降时保持可用；熔断在下游整体不健康时切断流量。顺序：有限重试 → 降级 → 熔断切走。

**Q8：如何避免供应商锁定？**

统一请求/响应协议 + Adapter 隔离差异；业务只用别名；路由与 Prompt 配置化；保留离线评测与影子流量验证替换质量。

---

## 面试回答加分点

1. **分层回答系统设计题**：目标 → 架构（网关/路由/Adapter/观测）→ 路由维度 → 稳定性（超时/重试/熔断/幂等）→ 灰度治理 → 安全与隐私。
2. 强调**模型别名**让“模型可变、业务接口稳定”。
3. 主动说**流式重试**和**已输出内容**的特殊处理，体现生产经验。
4. 灰度不只看 HTTP 5xx，还要看**解析失败、工具调用失败、拒答率**。
5. 降级有**安全边界**：高风险场景禁止随意换未验证小模型。
6. 提及**影子流量**与**稳定哈希分桶**，体现变更治理能力。
7. 日志**脱敏 + 可回放**：记版本与路由原因，不存完整 Prompt。
