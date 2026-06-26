# LLM 调用可靠性：超时、重试与熔断治理

## 主题选择记录

- **本次主题**：LLM 调用可靠性：超时、重试与熔断治理
- **主题序号**：第 50 篇
- **README 位置**：追加到 `## 目录` 表格末尾，归入「生产化与 LLMOps」卷；同时把知识库正文总数从 49 篇更新为 50 篇，并在平台 / 基建岗推荐与「卷次与篇次对应」中补充第 50 篇。
- **选择原因**：现有文档已经覆盖模型网关、多模型路由、成本缓存限流、故障演练、端到端延迟优化和高并发容量规划，但还没有单独从「一次 LLM 调用如何可靠完成」的角度展开超时预算、错误分类、重试退避、熔断半开、Fallback 与幂等保护。面试中经常会追问“模型超时要不要重试”“429 怎么处理”“重试会不会放大故障”“流式输出失败后怎么办”，这个主题能体现候选人是否具备生产级稳定性设计能力。
- **去重判断**：
  - `LLM模型网关与多模型路由` 重点讲统一调用层、Provider Adapter、路由与供应商治理；本文重点讲调用级可靠性策略和失败恢复边界。
  - `LLM应用故障演练与降级预案` 重点讲故障场景、演练流程、业务连续性和恢复复盘；本文重点讲在线请求发生异常时的超时、重试、熔断和 Fallback 决策。
  - `LLM应用端到端延迟优化` 重点讲性能拆解和延迟优化；本文重点讲可靠性与雪崩防护，关注“慢或失败时怎么安全处理”。

## 核心概念

**LLM 调用可靠性**指的是业务系统在调用大模型、Embedding、Rerank、内容审核、工具 API 等外部或内部智能服务时，能够在超时、限流、5xx、网络抖动、流式中断、输出格式错误等异常情况下，控制失败影响范围，避免请求雪崩，并给用户返回可解释、可恢复的结果。

面试中可以用一句话概括：**LLM 调用可靠性不是简单“失败就重试”，而是先设置调用预算，再区分错误类型，最后在重试、熔断、降级和安全失败之间做选择。**

一条生产级 LLM 调用链路通常包含这些可靠性控制点：

```text
接收请求
  → 计算总 deadline 与任务风险等级
  → 选择模型 / Provider
  → 设置分阶段 timeout
  → 调用模型或工具
  → 按错误类型决定是否重试
  → 更新健康状态与熔断器
  → 必要时触发 Fallback
  → 记录 trace、错误码、重试次数与降级原因
```

与普通 HTTP 服务相比，LLM 调用有几个特殊点：

| 特点 | 可靠性风险 | 设计重点 |
| --- | --- | --- |
| 响应慢且长尾明显 | 一个请求占用连接和线程很久 | 总 deadline、阶段 timeout、流式心跳 |
| 成本按 token 计费 | 盲目重试会直接放大成本 | 重试上限、预算控制、幂等与缓存 |
| 输出不完全确定 | 重试可能得到不同答案 | 结果校验、温度控制、版本记录 |
| 流式输出可能半途中断 | 用户收到半截内容，不能简单重放 | chunk 序号、可恢复提示、最终状态事件 |
| Agent 工具可能有副作用 | 重试可能重复发邮件、扣款、写库 | 幂等键、确认机制、补偿记录 |

可靠性治理的目标不是让所有请求都成功，而是让系统满足三个底线：

1. **不放大故障**：下游慢或不可用时，上游不能无限重试、堆积连接、打爆 Provider。
2. **不突破安全边界**：降级后仍要保留权限、审计、引用、人工审批等关键约束。
3. **不伪装成功**：质量或证据不足时，要明确低置信、排队、转人工或安全失败。

## 核心知识点

### 1. 先设计总 deadline，再拆分阶段 timeout

很多候选人只会说“设置超时时间”，但面试更关注是否会做**预算分配**。LLM 请求通常包含检索、重排、模型生成、工具调用、审核等阶段，如果每个阶段都独立设置较长 timeout，总耗时会失控。

推荐做法：

| 阶段 | 示例 timeout | 说明 |
| --- | ---: | --- |
| Query 改写 | 300ms～800ms | 超时可跳过，不应阻塞主链路 |
| 向量检索 | 500ms～1000ms | 超时可走关键词检索或缓存召回 |
| Rerank | 800ms～1500ms | 超时可使用初召排序，但要降低置信度 |
| 主模型生成 | 8s～30s | 按任务复杂度和输出长度设置 |
| 工具调用 | 1s～5s | 写操作必须有幂等键 |
| 内容审核 | 300ms～1000ms | 高风险场景不可跳过 |

代码示例：

```python
from dataclasses import dataclass
from time import monotonic

@dataclass
class Deadline:
    start: float
    total_ms: int

    def remaining_ms(self) -> int:
        elapsed = int((monotonic() - self.start) * 1000)
        return max(0, self.total_ms - elapsed)

def stage_timeout(deadline: Deadline, preferred_ms: int) -> int:
    # 中文注释：阶段超时不能超过总剩余预算，避免局部等待拖垮整条链路
    return min(preferred_ms, deadline.remaining_ms())

deadline = Deadline(start=monotonic(), total_ms=15000)
retrieval_timeout = stage_timeout(deadline, 800)
model_timeout = stage_timeout(deadline, 12000)
```

面试表达重点：**总 deadline 是用户体验和系统容量的边界，阶段 timeout 是资源分配策略。先有总预算，再谈每个环节等多久。**

### 2. 错误要分类，不是所有失败都能重试

LLM 调用常见错误可以分成四类：

| 错误类型 | 示例 | 是否重试 | 原因 |
| --- | --- | --- | --- |
| 可恢复瞬时错误 | 网络抖动、连接重置、少量 5xx | 可以有限重试 | 通常换时间片可恢复 |
| 限流与配额错误 | 429、TPM/RPM 超限 | 可退避或切路由 | 立即重试容易放大限流 |
| 业务不可重试错误 | 参数错误、鉴权失败、安全拒答 | 不重试 | 重试不会改变结果 |
| 副作用不确定错误 | 工具写入超时、扣款状态未知 | 默认不自动重试 | 需要查询状态或人工确认 |

代码示例：

```python
RETRYABLE_CODES = {"timeout", "connection_reset", "provider_5xx"}
BACKOFF_CODES = {"rate_limited", "quota_exceeded"}
NO_RETRY_CODES = {"bad_request", "unauthorized", "content_blocked"}

def classify_retry_action(error_code: str, has_side_effect: bool) -> str:
    # 中文注释：有副作用的未知失败要先查状态，不能直接重放请求
    if has_side_effect:
        return "check_state_before_retry"
    if error_code in RETRYABLE_CODES:
        return "retry_with_jitter"
    if error_code in BACKOFF_CODES:
        return "backoff_or_switch_provider"
    if error_code in NO_RETRY_CODES:
        return "fail_fast"
    return "fail_with_observable_error"
```

面试中不要只说“重试三次”。更好的表达是：**先判断错误是否可恢复、是否会产生副作用、是否已经向用户输出部分结果，再决定重试还是失败。**

### 3. 重试要有上限、退避、抖动和预算约束

LLM 服务容易因为重试造成雪崩：Provider 已经 429，上游还并发重试；流式请求已经输出一半，又重新发起导致用户看到重复内容；批处理失败后所有任务同时重试，形成尖峰。

可靠重试策略通常包含：

1. **最大尝试次数**：在线请求一般 1～2 次，批处理可更多但要排队。
2. **指数退避**：错误越多等待越久。
3. **随机抖动**：避免所有实例同一时间重试。
4. **总 deadline 约束**：剩余时间不足时不要重试。
5. **错误类型过滤**：只重试可恢复错误。
6. **幂等键**：涉及写操作或工具调用时必须具备。

代码示例：

```python
import random
import time

def retry_delay_ms(attempt: int, base_ms: int = 200, cap_ms: int = 2000) -> int:
    backoff = min(cap_ms, base_ms * (2 ** attempt))
    jitter = random.randint(0, 100)
    return backoff + jitter

def call_with_retry(call_fn, deadline: Deadline, max_attempts: int = 2):
    for attempt in range(max_attempts):
        try:
            timeout_ms = stage_timeout(deadline, 5000)
            if timeout_ms <= 0:
                raise TimeoutError("deadline_exceeded")
            return call_fn(timeout_ms=timeout_ms)
        except ProviderError as exc:
            # 中文注释：非可重试错误直接失败，避免把业务错误变成流量风暴
            if exc.code not in RETRYABLE_CODES or attempt == max_attempts - 1:
                raise
            delay = retry_delay_ms(attempt)
            if deadline.remaining_ms() <= delay:
                raise
            time.sleep(delay / 1000)
```

面试容易追问：**为什么重试次数不能太多？** 可以回答：LLM 请求成本高、耗时长、下游容量有限，过多重试会放大 token 成本和 Provider 压力，还可能造成答案不一致或副作用重复。

### 4. 熔断保护的是下游，也保护自己

熔断器用于在某个 Provider、模型、工具或检索服务持续异常时，短时间停止向它发送请求，避免错误扩散。

常见状态：

| 状态 | 含义 | 行为 |
| --- | --- | --- |
| Closed | 服务健康 | 正常放量 |
| Open | 错误率或超时率过高 | 拒绝请求或切备用 |
| Half-open | 冷却后探测恢复 | 只放少量探测流量 |

熔断指标要结合 LLM 特点：

- 错误率：5xx、连接失败、解析失败。
- 超时率：P95/P99 超过阈值。
- 429 比例：供应商限流或配额不足。
- 流式中断率：SSE / WebSocket 中途断开。
- 结构化输出失败率：JSON Schema 不合法、字段缺失。
- 安全拒答异常波动：可能是策略变更或模型版本变化。

示例：

```python
from enum import Enum

class CircuitState(str, Enum):
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"

def should_open_circuit(total: int, errors: int, p95_ms: int) -> bool:
    # 中文注释：样本量太小时不熔断，避免偶发错误导致误切流量
    if total < 50:
        return False
    error_rate = errors / total
    return error_rate > 0.25 or p95_ms > 20000

def allow_request(state: CircuitState, probe_ratio: float = 0.05) -> bool:
    if state == CircuitState.CLOSED:
        return True
    if state == CircuitState.OPEN:
        return False
    # 中文注释：半开状态只允许少量请求探测恢复情况
    return random.random() < probe_ratio
```

面试表达重点：**熔断不是“服务挂了才切”，而是在错误率、超时率和限流信号持续恶化时提前止损；恢复时不能一把全量切回，要半开探测。**

### 5. Fallback 要按风险分层，不能只会“换小模型”

Fallback 是主链路不可用时的兜底策略。常见方案包括：

| Fallback 类型 | 适用场景 | 风险 |
| --- | --- | --- |
| 切备用 Provider | 主供应商 5xx 或超时 | 模型差异导致输出风格或质量变化 |
| 切小模型 | 低风险摘要、分类、润色 | 复杂推理质量下降 |
| 缩短上下文 | token 超预算或延迟异常 | 可能丢失关键证据 |
| 使用缓存结果 | 热点 FAQ、固定知识 | 权限、版本和时效性风险 |
| 转异步排队 | 长文档处理、批量生成 | 用户等待，需要进度通知 |
| 安全失败 / 转人工 | 金融、医疗、法务、写操作 | 体验下降，但更安全 |

一个实用决策表：

```python
def choose_fallback(task_risk: str, evidence_required: bool, provider_down: bool) -> str:
    # 中文注释：高风险任务优先安全失败，不用弱答案伪装成功
    if task_risk in {"high", "critical"}:
        return "human_review_or_safe_fail"
    if evidence_required and provider_down:
        return "answer_only_from_cached_evidence_with_notice"
    if provider_down:
        return "switch_backup_provider"
    return "short_answer_or_async_queue"
```

面试中可以强调：**Fallback 的底线是质量和安全边界不能被绕过。RAG 场景证据不足就不要编，Agent 写操作不确定就不要自动重放。**

### 6. 流式输出失败要单独处理

流式调用的难点在于“已经给用户输出了一部分”。如果中途失败，不能像普通请求一样无感重试，否则用户可能看到重复片段或前后不一致。

设计要点：

1. 每个 chunk 带序号、request_id 和事件类型。
2. 服务端记录最终状态：completed、cancelled、failed、partial。
3. 中断后明确告诉用户“回答未完成”，允许重新生成或转异步。
4. 如果要续写，要把已输出内容作为上下文，并声明是继续生成。
5. 对结构化 JSON 流，必须等最终校验通过后才触发下游动作。

示例事件：

```json
{"event":"chunk","request_id":"r-001","seq":1,"text":"根据现有资料，"}
{"event":"chunk","request_id":"r-001","seq":2,"text":"可以分三点来看。"}
{"event":"error","request_id":"r-001","code":"stream_interrupted","retryable":true}
```

面试表达重点：**流式失败不是简单 HTTP 失败，它已经产生了用户可见副作用，所以要有最终状态事件和可解释恢复方式。**

### 7. Agent 工具调用重试必须结合幂等和补偿

Agent 场景最容易在面试中被追问，因为工具调用可能真的改变业务数据。比如创建工单、发送邮件、修改配置、扣款、审批流提交，这些动作不能因为模型或网络超时就重复执行。

可靠设计：

- 每次工具调用生成 `idempotency_key`。
- 工具服务以幂等键做去重或返回上次结果。
- 调用超时后先查询状态，不直接重试。
- 写操作进入 outbox 或操作日志，便于补偿。
- 高风险动作需要用户确认或人工审批。

代码示例：

```python
def execute_tool(tool_name: str, args: dict, conversation_id: str) -> dict:
    idempotency_key = f"{conversation_id}:{tool_name}:{hash_args(args)}"
    try:
        return tool_client.call(
            name=tool_name,
            args=args,
            idempotency_key=idempotency_key,
            timeout_ms=3000,
        )
    except TimeoutError:
        # 中文注释：写操作超时后先查幂等键状态，避免重复执行副作用
        existing = tool_client.query_by_idempotency_key(idempotency_key)
        if existing:
            return existing
        raise PendingToolState(idempotency_key)
```

面试表达重点：**读操作可以有限重试，写操作必须先保证幂等；状态未知时进入待确认或补偿流程。**

### 8. 可观测性要记录“为什么这么处理”

可靠性策略如果没有日志和指标，线上很难解释问题。一次 LLM 调用至少记录：

- request_id、tenant_id、user_id_hash、task_type。
- model_alias、provider、model_version、prompt_version。
- total_deadline_ms、各阶段耗时、TTFT、总耗时。
- error_code、retry_count、backoff_ms、circuit_state。
- fallback_type、fallback_reason、risk_level。
- token 用量、估算成本、是否命中缓存。
- 是否流式中断、是否输出结构化校验失败。

注意：**不要直接记录完整 Prompt、用户隐私、密钥或未脱敏业务数据。**

面试中可以补一句：这些字段不仅用于排查故障，也用于评估重试策略是否过激、熔断阈值是否合理、Fallback 是否影响质量和成本。

## 高频面试问题与标准答案

**Q1：线上 LLM 调用超时，你会怎么处理？**

我会先区分是哪个阶段超时，比如检索、rerank、模型生成还是工具调用。设计上会有总 deadline 和阶段 timeout，阶段超时不能无限等待。如果是低风险任务，可以走缓存、缩短上下文、切备用模型或转异步；如果是高风险任务，比如法务、金融、写操作，我更倾向于安全失败或转人工。关键是不能因为超时就让模型凭空回答，也不能无限重试拖垮系统。

**Q2：模型接口失败时是不是直接重试三次？**

不会直接这么做。我会先看错误类型：网络抖动、少量 5xx 可以有限重试；429 要退避或切 Provider；参数错误、鉴权失败、安全拒答不重试；如果涉及写操作或工具副作用，超时后要先查状态或依赖幂等键，不能盲目重放。在线请求一般最多重试 1～2 次，并且受总 deadline 限制。

**Q3：如何避免重试导致雪崩？**

首先重试要有上限，不能所有错误都重试；其次要用指数退避和随机抖动，避免所有实例同时打下游；再者要结合熔断器，当某个 Provider 错误率或超时率持续升高时停止放量，切备用或降级。还要按租户和任务优先级限流，防止低优先级请求把核心链路拖垮。

**Q4：429 限流错误应该怎么处理？**

429 通常表示供应商 RPM、TPM 或配额触顶，立即重试意义不大，甚至会更糟。我会读取响应里的 retry-after 或根据本地策略退避；如果业务要求实时性，可以切备用 Provider 或小模型；如果是低优先级任务，可以排队或转异步。同时要把 429 作为容量和预算指标看，长期高频出现说明限流、路由或配额规划有问题。

**Q5：熔断和降级有什么区别？**

熔断是保护机制，发现某个下游持续不健康后，短时间停止把请求发给它；降级是业务策略，在能力下降时用备用方案维持可用，比如短答案、缓存、备用模型、异步或转人工。熔断解决“不要继续打坏的下游”，降级解决“用户请求接下来怎么处理”。通常熔断触发后会配合降级或路由切换。

**Q6：熔断后如何恢复？**

不能熔断结束就全量切回。比较稳妥的做法是进入半开状态，只放少量探测流量，观察错误率、P95、429、流式中断率和结构化输出失败率。如果指标稳定，再逐步提高流量；如果探测失败，继续保持熔断。这样可以避免服务刚恢复又被全量流量打挂。

**Q7：Fallback 是不是切到便宜小模型就可以？**

不是。Fallback 要按任务风险分层。低风险摘要、润色可以切小模型或短输出；企业知识库问答可以用缓存或关键词检索兜底，但要说明证据不足；高风险决策或有副作用的 Agent 操作，宁可转人工或安全失败，也不能用弱模型继续自动执行。我的原则是降级可以牺牲体验，但不能突破权限、证据和审批边界。

**Q8：流式输出中断怎么处理？**

流式中断已经让用户看到部分内容，所以不能简单无感重试。我会给每个 chunk 带 request_id 和序号，服务端记录最终状态。如果中断，前端显示“回答未完成”，允许用户重新生成或继续生成；如果是结构化输出或工具执行，必须等最终校验完成后才能触发下游动作，不能拿半截 JSON 去执行业务。

**Q9：Agent 调工具失败后如何重试？**

先区分读操作和写操作。读操作可以有限重试；写操作必须有幂等键，比如 conversation_id、tool_name 和参数哈希组成的 idempotency_key。调用超时后要先按幂等键查询是否已经执行成功，如果状态未知，就进入待确认、补偿或人工处理，而不是让 Agent 自己重复调用。这样可以避免重复发邮件、重复创建工单或重复扣款。

**Q10：如何设计一次 LLM 调用的可靠性监控？**

我会记录 request_id、租户、任务类型、模型别名、实际 Provider、Prompt 版本、各阶段耗时、TTFT、总耗时、错误码、重试次数、退避时间、熔断状态、Fallback 原因、token 和成本。指标上关注错误率、超时率、429、P95/P99、流式中断率、结构化解析失败率。这样出了问题可以回答“慢在哪里、为什么重试、为什么降级、影响了哪些用户”。

**Q11：如果备用模型质量比主模型差，切换时怎么保证效果？**

首先要按任务风险决定是否允许切换，不是所有场景都能切。允许切换的场景要提前用评测集验证备用模型的最低质量门槛，并限制它的能力范围，比如只做摘要草稿或低风险问答。线上切换时记录 fallback_reason 和 model_version，必要时加低置信提示或人工确认。高风险场景如果备用模型达不到标准，就应该安全失败或转人工。

**Q12：面试官问“你们怎么设置超时时间”，怎么回答更像做过线上系统？**

我不会只报一个固定数字，而是说我们先按产品 SLA 设置总 deadline，再按链路拆阶段预算，比如检索 800ms、rerank 1.2s、模型生成 12s，并且阶段 timeout 不能超过总剩余时间。不同任务会分层：实时客服更关注 TTFT 和总耗时，长文档处理可以转异步；高风险任务即使超时也不能跳过审核或权限过滤。超时时间最终要结合线上 P95/P99、错误率和用户体验持续调整。
