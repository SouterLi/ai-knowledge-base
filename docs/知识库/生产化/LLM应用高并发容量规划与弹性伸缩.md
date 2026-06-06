# LLM 应用高并发容量规划与弹性伸缩

## 主题选择记录

- **本次主题序号**：第 36 篇。
- **README 位置**：追加到知识库目录表第 35 篇之后，归入「生产化与 LLMOps」卷。
- **选择原因**：AI 应用面试中，平台化、企业内部助手、智能客服、RAG/Agent 落地场景经常追问“并发上来怎么办”“怎么估机器”“怎么限流扩容”“怎么保证租户隔离”。现有文档已覆盖成本缓存限流、流式异步、端到端延迟、私有化推理，但还没有单独讲高并发容量规划和弹性伸缩。
- **与已有主题的边界**：第 11 篇侧重成本、缓存和限流策略；第 12 篇侧重流式/异步任务形态；第 24 篇侧重端到端延迟优化；第 28 篇侧重私有化推理服务。本文重点放在**容量估算、削峰背压、资源隔离、自动扩缩容和面试架构表达**。

## 核心概念

### 1. 高并发不是只看 QPS

LLM 应用的并发压力通常由四类指标共同决定：

| 指标 | 含义 | 面试重点 |
| --- | --- | --- |
| QPS / RPM | 请求数量 | 入口网关、业务 API、检索服务压力 |
| TPM | 每分钟 token 数 | 模型网关和推理服务容量核心 |
| 并发连接数 | 同时进行中的请求 | 流式响应、长任务、连接池容量 |
| 队列等待时间 | 请求进入执行前的排队耗时 | 是否需要削峰、异步化、扩容 |

普通 Web 系统常按 QPS 估容量，但 LLM 应用必须同时看 **token 吞吐、上下文长度、输出长度、流式连接时长和上游限额**。

### 2. 容量规划

容量规划是把业务目标转换为系统资源：

```text
业务峰值请求量 → token 分布 → 链路耗时 → 关键瓶颈 → 副本数/队列/限流阈值
```

面试中不能只说“加机器”，要能讲清楚：

- 峰值 QPS、平均和 P95 token 长度；
- RAG 检索、rerank、模型推理、工具调用各自耗时；
- 上游模型 RPM/TPM 限制；
- 需要多少 API 副本、Worker 副本、模型副本、向量库实例；
- 超出容量后如何排队、降级或拒绝。

### 3. 削峰、背压与降级

- **削峰**：用队列、异步任务、批处理把瞬时流量摊平。
- **背压**：下游处理不过来时，上游主动放慢、排队或拒绝，避免雪崩。
- **降级**：在容量紧张时关闭非核心链路，例如缩短上下文、跳过 rerank、切小模型、转异步或返回简答。

LLM 应用尤其要避免“无限重试 + 无限排队”，因为这会同时放大延迟和 token 成本。

### 4. 弹性伸缩

弹性伸缩不是简单按 CPU 扩容。LLM 应用更常见的伸缩信号包括：

- API 层：QPS、连接数、P95 延迟、线程/协程池水位；
- 队列层：队列长度、最老消息等待时间、消费速率；
- 模型层：TPM、tokens/s、TTFT、TPOT、GPU 利用率、KV Cache 水位；
- 检索层：向量检索 QPS、P95 查询耗时、连接池等待。

### 5. 资源隔离

生产系统必须按租户、用户等级、任务类型隔离资源：

- 高优先级在线问答优先于离线批量总结；
- 付费租户和免费租户额度不同；
- 写操作 Agent 不能和普通聊天共享无限工具调用资源；
- RAG 检索、rerank、模型调用都要有独立超时和并发上限。

## 核心知识点

### 1. 推荐的高并发架构

```text
客户端
  ↓
API 网关：鉴权、租户识别、限流、熔断
  ↓
应用服务：Prompt 组装、RAG 编排、工具编排、任务状态
  ↓
任务队列：削峰、优先级、重试、死信
  ↓
Worker 池：RAG Worker / Agent Worker / 批处理 Worker
  ↓
模型网关：模型路由、TPM 控制、重试、降级
  ↓
模型服务 / 第三方模型 / 向量库 / 业务系统
```

在线短问答可以同步或流式返回；长文档总结、批量抽取、复杂 Agent 任务应进入异步队列，返回 `task_id`。

### 2. 容量估算的基本公式

面试时可以用一套“够工程化”的估算方法：

```text
峰值输入 TPM = 峰值 QPS × 平均输入 token × 60
峰值输出 TPM = 峰值 QPS × 平均输出 token × 60
并发请求数 ≈ 峰值 QPS × P95 端到端耗时
所需模型副本 ≈ 峰值 tokens/s ÷ 单副本 tokens/s × 安全系数
```

示例代码：

```python
import math


def estimate_capacity(qps: float, input_tokens: int, output_tokens: int,
                      p95_latency_sec: float, replica_tokens_per_sec: int,
                      safety_factor: float = 1.3) -> dict:
    total_tpm = qps * (input_tokens + output_tokens) * 60
    inflight = qps * p95_latency_sec
    required_tokens_per_sec = total_tpm / 60
    replicas = math.ceil(required_tokens_per_sec / replica_tokens_per_sec * safety_factor)

    return {
        "total_tpm": int(total_tpm),
        "inflight_requests": math.ceil(inflight),
        "model_replicas": replicas,  # 中文注释：副本数要留安全系数，防止峰值和长尾请求打满
    }


print(estimate_capacity(
    qps=20,
    input_tokens=1200,
    output_tokens=600,
    p95_latency_sec=8,
    replica_tokens_per_sec=2500,
))
```

这个估算不是最终答案，还要结合真实压测：长短问题比例、RAG top_k、是否流式、工具调用耗时、模型供应商限流。

### 3. 队列削峰与优先级

高并发下，队列不是“所有请求都塞进去”，而是要设计优先级和超时：

| 队列类型 | 适用场景 | 关键控制 |
| --- | --- | --- |
| 在线高优先级队列 | 用户等待中的聊天、问答 | 短超时、保交互体验 |
| 普通任务队列 | 文档总结、批量抽取 | 可排队、可重试 |
| 低优先级队列 | 离线评估、批量 embedding | 限速、空闲时执行 |
| 死信队列 | 多次失败任务 | 人工排查、补偿处理 |

```python
def choose_queue(task_type: str, tenant_level: str) -> str:
    if task_type == "chat" and tenant_level in {"vip", "internal_core"}:
        return "online_high"
    if task_type in {"batch_summary", "document_extract"}:
        return "offline_normal"
    return "online_normal"  # 中文注释：默认队列要有容量上限，不能无限堆积
```

### 4. 背压策略

背压要沿链路逐层传递：

1. 模型网关发现 TPM 接近上限；
2. Worker 降低消费速率或暂停低优先级队列；
3. API 层对新请求返回排队、降级或 429；
4. 前端展示预计等待时间或建议稍后重试。

常见策略：

- 队列长度超过阈值：低优先级任务延迟执行；
- 最老消息等待超过阈值：触发扩容或拒绝新任务；
- 模型 TPM 达到阈值：切小模型、缩短上下文、减少 max_tokens；
- 下游业务 API 异常：熔断工具调用，返回可解释失败。

### 5. 自动扩缩容指标

Kubernetes HPA 适合 CPU/内存型服务，但 LLM Worker 更适合结合队列和自定义指标：

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: llm-worker-scaler
spec:
  scaleTargetRef:
    name: llm-worker
  minReplicaCount: 2
  maxReplicaCount: 30
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus:9090
        metricName: queue_oldest_wait_seconds
        threshold: "10"
        query: max(llm_queue_oldest_wait_seconds{queue="online_normal"})
```

面试回答要强调：**只按 CPU 扩容容易误判**。比如 Worker 在等模型返回时 CPU 不高，但用户已经排队很久，所以要看队列等待、TTFT、TPM、tokens/s。

### 6. 租户级限流与资源配额

限流维度建议至少包括：

- `tenant_id`：租户级总额度；
- `user_id`：防止单用户打爆租户额度；
- `model`：不同模型成本和容量不同；
- `task_type`：聊天、RAG、Agent、批处理分开；
- `token_budget`：按预估输入输出 token 控制，而不只按请求数。

```python
def allow_request(tenant_id: str, task_type: str, estimated_tokens: int) -> bool:
    quota = quota_repo.get_quota(tenant_id, task_type)
    used = usage_repo.current_window_usage(tenant_id, task_type)

    if used.requests >= quota.rpm:
        return False
    if used.tokens + estimated_tokens > quota.tpm:
        return False  # 中文注释：LLM 限流必须看 token，否则长上下文请求会绕过 QPS 控制
    return True
```

### 7. 压测方法

压测要尽量模拟真实业务分布，而不是只用短 prompt：

- 20% 短问题、60% 普通 RAG、20% 长上下文或复杂 Agent；
- 同时测同步、流式、异步任务；
- 记录 QPS、TPM、TTFT、TPOT、P95/P99、错误率、队列等待；
- 找到拐点：延迟开始陡增、错误率上升、队列持续积压、GPU/KV Cache 接近上限；
- 根据拐点反推限流阈值和扩容阈值。

### 8. 常见故障模式

| 故障 | 表现 | 处理 |
| --- | --- | --- |
| 队列无限增长 | 用户等待越来越久 | 设置最大等待、扩容、低优先级丢弃 |
| 重试风暴 | 429/超时后请求更多 | 指数退避、重试预算、熔断 |
| 长上下文拖垮吞吐 | TTFT 变长、TPM 打满 | 上下文压缩、分层检索、限制 max_tokens |
| 租户互相影响 | 某租户打爆全局容量 | 租户级队列、配额、隔离池 |
| 扩容无效 | 副本增加但延迟不降 | 瓶颈在模型限额、向量库或下游工具 |

## 高频面试问题与标准答案

**Q1：如果一个 LLM 应用从 100 用户增长到 10 万用户，你会怎么做容量规划？**

我会先拆业务流量，而不是直接按用户数估机器。先看 DAU、峰值并发、每次会话请求数，再统计平均和 P95 输入输出 token。然后把链路拆成网关、应用服务、RAG 检索、rerank、模型推理、工具调用，分别压测出瓶颈。模型侧重点看 TPM、tokens/s、TTFT、TPOT；应用侧看 QPS、连接数、队列等待。最后根据压测拐点设置副本数、限流阈值、队列长度和降级策略。

**Q2：LLM 应用为什么不能只按 QPS 限流？**

因为两个请求的成本可能差很多。一个 200 token 的分类请求和一个 8K 上下文的 RAG 问答，对模型吞吐、KV Cache、延迟和费用的压力完全不同。所以我会同时按 RPM、TPM、并发数、任务类型和租户预算限流，尤其要按预估 token 做准入控制。

**Q3：高峰期请求太多，你会直接扩容吗？**

不会只说扩容。我会先判断瓶颈在哪里：如果 API CPU 高，扩 API 副本；如果队列等待长，扩 Worker 或做削峰；如果模型 TPM 打满，扩模型副本、切小模型或压缩上下文；如果是向量库慢，就优化索引、连接池或扩检索实例。扩容要和限流、降级、缓存一起做，否则可能只是把压力转移到下游。

**Q4：队列在 LLM 应用里怎么设计？**

我会按任务类型和优先级拆队列：在线聊天高优先级，文档总结普通队列，离线 embedding 低优先级。每个队列都有最大长度、最大等待时间、重试次数和死信队列。Worker 消费时也要有租户配额，避免某个大租户把所有 Worker 占满。

**Q5：自动扩缩容看哪些指标？**

API 层可以看 CPU、连接数、P95 延迟；Worker 层更应该看队列长度、最老消息等待时间、消费速率；模型层要看 TPM、tokens/s、TTFT、TPOT、GPU 利用率和 KV Cache 水位。我的经验是，LLM 应用只看 CPU 很容易误判，因为很多时间是在等模型或下游返回。

**Q6：如何避免一个租户影响其他租户？**

入口就识别 `tenant_id`，做租户级 RPM/TPM/预算控制；队列按租户或优先级隔离；模型网关保留核心租户额度；缓存 key、检索过滤和任务状态都带租户范围。对于特别大的客户，可以给独立队列、独立模型池或独立向量库实例。

**Q7：流量突增但模型供应商限额不能马上提高，怎么办？**

先保护核心链路：高优先级用户保额度，低优先级任务排队或延迟；同时缩短上下文、降低 `max_tokens`、启用缓存、简单问题走小模型，必要时关闭 rerank 或复杂 Agent 步骤。对用户侧要给明确提示，比如“当前排队中”或转异步任务，而不是让请求一直卡住。

**Q8：你如何压测一个 RAG + Agent 系统？**

我会构造接近真实的混合流量：短问答、普通 RAG、长文档、多工具 Agent 都要覆盖。压测时记录每段耗时，包括查询改写、检索、rerank、模型生成、工具调用和最终汇总。重点看 P95/P99、队列等待、模型 TTFT/TPOT、错误率和成本。最后找系统拐点，用它来确定限流、扩容和降级阈值。

**Q9：扩容后延迟还是很高，你怎么排查？**

我会先看链路分段指标，确认慢在排队、检索、模型 prefill、decode 还是工具调用。如果 API 副本扩了但队列等待不降，可能 Worker 或模型是瓶颈；如果 Worker 增加但模型 429 增加，说明上游限额到了；如果 TTFT 高，可能是长上下文或 prefill 压力；如果只有部分租户慢，要看是否命中配额或热点数据。核心是用指标定位瓶颈，不盲目加机器。

**Q10：面试中让你设计 1000 并发智能客服，你会怎么回答？**

我会按“入口限流、应用编排、队列削峰、模型网关、RAG 检索、观测降级”来讲。入口做鉴权和租户限流，在线问题走流式，长任务异步化；RAG 检索和 rerank 设置超时；模型网关按 TPM 做路由和降级；高峰期用缓存、小模型和上下文压缩保护核心体验；最后用 TTFT、P95、队列等待、TPM、错误率来监控和扩容。
