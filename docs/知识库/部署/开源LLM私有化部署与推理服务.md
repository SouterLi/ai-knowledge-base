# 开源大模型私有化部署与推理服务

## 核心概念

### 1. 私有化部署

将模型权重、推理服务、向量库与业务系统部署在企业自有云/专有云/内网。目标：数据合规、成本可控、可定制 SLA、离线可用。  
**注意**：私有化不必然更便宜，依赖调用量、GPU 利用率与运维成熟度。

### 2. 推理服务

加载权重 → 调度请求 → 自回归生成 → 同步/流式返回。能力包括：量化权重、动态批处理、KV Cache、OpenAI 兼容 API、指标（QPS、TTFT、TPOT）。

### 3. 推理引擎选型

| 引擎 | 特点 | 场景 |
| --- | --- | --- |
| vLLM | PagedAttention、连续批处理、OpenAI API | 通用在线服务 |
| TGI | HF 生态 | HF 模型服务化 |
| TensorRT-LLM | NVIDIA 深度优化 | 大规模 GPU |
| llama.cpp | CPU/边缘、量化 | 轻量本地 |
| Ollama | 易用 | 开发原型，非高并发生产核心 |

### 4. 显存与 KV Cache

显存 ≈ 模型权重 + KV Cache（随 batch、上下文长度增）+ 运行时开销。  
7B FP16 权重约 14GB，不代表 24GB 卡就能高并发长上下文。

### 5. TTFT 与 TPOT

- **TTFT**：首 token 时间（排队 + prefill + 网络）。  
- **TPOT**：每输出 token 耗时（decode 吞吐）。  

交互对话重 TTFT/流式；批量任务重吞吐。

---

## 核心知识点

### 1. 分层架构

```text
业务 / RAG / Agent → 模型网关 → 推理 API → vLLM/TGI/TRT-LLM → GPU → 监控告警
```

业务只认 **model_alias**；网关负责鉴权、限流、路由、灰度、审计。

### 2. 显存粗算

```python
def estimate_weight_memory_gb(params_billion: float, bytes_per_param: float) -> float:
    # 中文注释：仅权重估算；生产需加 KV Cache 与 20%~30% 冗余
    return params_billion * 1e9 * bytes_per_param / (1024 ** 3)

print(round(estimate_weight_memory_gb(7, 2), 1))   # FP16 约 13GB
print(round(estimate_weight_memory_gb(7, 0.5), 1))  # INT4 约 3.3GB
```

### 3. 量化取舍

INT8/INT4 降显存与成本，可能损复杂推理、代码、结构化输出；必须用**业务评测集**对比后再上线。

### 4. 动态批处理与队列

Continuous batching 提高 GPU 利用率，但可能增加排队与 TTFT。配置 `max_num_seqs`、`max_model_len`、超时与优先级。

```yaml
serving:
  max_model_len: 8192
  max_num_seqs: 64
  request_timeout_ms: 30000
  queue_policy:
    max_wait_ms: 3000
    reject_when_queue_full: true
```

### 5. OpenAI 兼容 API

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://llm-gateway.internal/v1",
    api_key="internal-token",
)
resp = client.chat.completions.create(
    model="private_chat",  # 中文注释：别名由网关路由到实际推理实例
    messages=[{"role": "user", "content": "总结本季度投诉主因"}],
    temperature=0.2,
    max_tokens=512,
)
print(resp.choices[0].message.content)
```

### 6. 流式 SSE

客户端断开应取消后端生成；流式也要采样日志；工具/JSON 场景处理半截输出。

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()

async def token_stream():
    # 中文注释：生产应转发推理引擎并处理超时与取消
    for tok in ["私", "有", "化"]:
        yield f"data: {tok}\n\n"
    yield "data: [DONE]\n\n"

@app.get("/v1/chat/stream")
async def stream():
    return StreamingResponse(token_stream(), media_type="text/event-stream")
```

### 7. 高可用与降级

多副本、含真实推理的健康检查、熔断、按队列切小模型、缩短上下文、灰度与回滚。

```python
def choose_model(tenant_level: str, queue_ms: int) -> str:
    # 中文注释：排队过长时降级到 fast 模型，企业高价值租户保 strong
    if tenant_level == "enterprise":
        return "private_chat_strong"
    if queue_ms > 2000:
        return "private_chat_fast"
    return "private_chat_balanced"
```

### 8. 监控指标

业务（QPS、成功率）、延迟（TTFT、TPOT、P95）、Token（in/out、tokens/s）、资源（GPU 利用率、显存、OOM）、质量（结构化成功率、坏 Case）。

### 9. 压测与私有化交付

压测用真实 token 分布（长短输入、流式/非流式）；逐步升并发找拐点。  
交付前确认：GPU/驱动、镜像离线包、许可证、日志脱敏、验收 SLA、回滚方案。

---

## 高频面试问题与标准答案

**Q1：7B 模型如何部署到内网？**  
选模型（许可证、中文、工具调用）→ 引擎（vLLM/TGI）→ 显存+KV+并发估算 → OpenAI API + 网关 → 压测 TTFT/P95 → 监控灰度回滚。

**Q2：能加载但并发就 OOM？**  
权重仅占一部分；KV Cache 随并发与上下文增长；查 max_seqs、max_model_len、量化、加副本或限流。

**Q3：为何 vLLM 适合在线？**  
PagedAttention + continuous batching 提高利用率；OpenAI 兼容；极致性能可看 TRT-LLM，边缘看 llama.cpp。

**Q4：量化收益与风险？**  
省显存/成本；可能损质量；必须业务评测，不能只看能否启动。

**Q5：如何优化 TTFT？**  
拆排队 vs prefill；限流扩容、prefix caching、压缩 RAG 上下文、网关与网络优化。

**Q6：私有化一定比云 API 便宜？**  
不一定；小流量或极高质量要求可能云更划算；算 TCO（硬件、运维、利用率、失败成本）。

**Q7：私有模型高可用？**  
多副本、推理级健康检查、熔断、降级路由、灰度、监控 OOM/队列。

**Q8：如何接入 RAG/Agent？**  
经网关统一别名；验证 function calling 与多轮稳定性；业务不绑死单一引擎。

**Q9：如何压测？**  
真实长度分布；看 QPS、TTFT、TPOT、P99、OOM、GPU 水位；定限流与副本数。

**Q10：离线交付确认什么？**  
硬件、网络、许可证、脱敏、验收指标、离线包与回滚、运维手册。

---

## 面试回答加分点

1. **六步开放题**：目标 → 模型 → 资源 → 引擎 → 治理 → 验证上线。  
2. **反对**：只按权重大小选卡、业务直连 vLLM、盲目量化、只看公开榜。  
3. **混合云路由**：敏感走私有、通用走云，网关统一。  
4. **KV Cache 讲清**：解释「加载成功≠能扛并发」。  
5. **流式与成本**：断开取消生成省 GPU。  
6. **许可证**：开源≠可任意商用与再分发。  
7. **一句话**：私有化是把模型变成**可治理的生产服务**，不是跑通 Ollama。
