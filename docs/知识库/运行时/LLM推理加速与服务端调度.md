# LLM 推理加速与服务端调度

## 主题选择记录

- **本次主题序号**：第 41 篇。
- **README 位置**：追加到知识库目录表第 40 篇之后，归入「运行时与性能」卷。
- **选择原因**：AI 应用开发面试中，除了会问“接口为什么慢”“如何做高并发”，还经常追问模型推理服务内部的瓶颈，例如 prefill/decode、KV Cache、连续批处理、投机解码、Prefix Cache 和 GPU 利用率。现有文档已覆盖端到端延迟、容量规划和私有化部署，但缺少一篇从推理引擎调度视角展开的专题。
- **与已有主题的边界**：第 12 篇侧重流式响应与异步任务架构；第 24 篇侧重端到端链路延迟；第 28 篇侧重私有化部署与推理服务选型；第 36 篇侧重高并发容量规划。本文重点放在**模型推理内部阶段、KV Cache 显存管理、批处理调度、投机解码与线上调参取舍**。

## 核心概念

### 1. 推理阶段：Prefill 与 Decode

LLM 在线推理通常分成两个阶段：

| 阶段 | 做什么 | 主要影响指标 |
| --- | --- | --- |
| Prefill | 处理输入 prompt，计算所有输入 token 的隐藏状态并生成首 token 前的 KV Cache | TTFT、输入 token 长度、长上下文成本 |
| Decode | 自回归逐 token 生成，每一步都读取历史 KV Cache 并产出下一个 token | TPOT、tokens/s、完整生成耗时 |

面试里要把“模型慢”拆清楚：**TTFT 高通常看排队和 prefill；TPOT 高通常看 decode 吞吐、批处理和 GPU 利用率**。

### 2. KV Cache

KV Cache 是 Transformer 推理时缓存每层 attention 的 Key/Value，避免生成第 N 个 token 时反复计算前 N-1 个 token。它能显著加速 decode，但会占用大量显存：

```text
KV Cache 大小 ≈ batch_size × seq_len × layer_count × hidden_size × 2(K/V) × bytes
```

所以“模型权重能加载”不等于“能支撑高并发长上下文”。长上下文、多并发和大 batch 会一起推高 KV Cache 水位。

### 3. 连续批处理

传统 batch 往往等一批请求全部结束后再处理下一批；LLM 请求输出长度差异很大，容易让短请求等待长请求。**Continuous Batching（连续批处理）**会在每个 decode step 动态加入新请求、移除已完成请求，提高 GPU 利用率。

它的核心取舍是：吞吐提升，但如果队列策略不合理，可能增加排队时间和 TTFT。

### 4. Prefix Cache 与 Prompt Cache

Prefix Cache 是推理引擎复用相同前缀 prompt 的 KV Cache，常见于固定系统提示词、固定工具说明、固定 RAG 模板的场景。它和业务层“问答缓存”不同：

- Prefix Cache 复用的是**模型中间状态**；
- 业务问答缓存复用的是**最终答案**；
- Prefix Cache 命中要求前缀 token 完全一致或满足引擎支持的匹配规则。

### 5. 投机解码

投机解码（Speculative Decoding）用一个小模型先快速生成候选 token，再由大模型批量验证。若候选正确，大模型一次接受多个 token；若不正确，则回退到大模型结果。它的目标是降低每个输出 token 的平均耗时。

它不是“用小模型替代大模型”，而是**小模型提案、大模型验收**。收益依赖小模型命中率、验证开销、输出长度和部署成本。

## 核心知识点

### 1. 推理指标要分层看

| 指标 | 含义 | 常见优化方向 |
| --- | --- | --- |
| TTFT | 首 token 时间 | 减少排队、缩短输入、Prefix Cache、prefill 优化 |
| TPOT | 每输出 token 耗时 | 连续批处理、投机解码、量化、并行策略 |
| tokens/s | 单副本或集群吞吐 | 批处理、GPU 利用率、模型并行 |
| GPU 利用率 | 计算资源使用情况 | batch 调度、减少 CPU/网络瓶颈 |
| KV Cache 使用率 | 显存中 KV Cache 占用 | 限制上下文、PagedAttention、并发准入 |
| 队列等待 | 请求进入推理前等待时间 | 优先级、限流、扩容、拒绝策略 |

面试回答建议先说“我会同时看 TTFT、TPOT、队列等待和 KV Cache 水位”，体现你能区分用户体验、吞吐和资源瓶颈。

### 2. KV Cache 为什么会成为瓶颈

长上下文和高并发会让 KV Cache 成为比模型权重更动态的显存消耗。常见故障是：

- 7B/14B 模型能启动，但并发上来后 OOM；
- 长文档 RAG 请求挤占 KV Cache，短问答 TTFT 变差；
- `max_model_len` 配得过大，导致可用并发下降；
- 流式请求断开后没有及时取消生成，KV Cache 长时间占用。

```python
def estimate_kv_cache_gb(concurrency: int, seq_len: int, kv_per_token_kb: float) -> float:
    # 中文注释：用于面试中的粗估，真实大小还取决于层数、head 数、精度和引擎实现
    return concurrency * seq_len * kv_per_token_kb / 1024 / 1024


print(round(estimate_kv_cache_gb(
    concurrency=64,
    seq_len=8192,
    kv_per_token_kb=512,
), 1))
```

工程上要做准入控制：按预估输入 token、最大输出 token、当前 KV Cache 水位决定接收、排队、降级还是拒绝。

### 3. PagedAttention 与显存碎片

PagedAttention 的思路类似操作系统分页：把 KV Cache 切成块管理，避免每个请求都申请连续大块显存。它能改善显存碎片和动态 batch 场景下的利用率，是 vLLM 适合在线服务的重要原因之一。

面试表达要点：

1. 它解决的是 KV Cache 管理效率问题；
2. 它不改变模型本身能力；
3. 它提升吞吐和并发，但仍受总显存、上下文长度和输出长度限制。

### 4. 连续批处理的调参取舍

常见参数包括：

```yaml
serving:
  max_num_seqs: 64
  max_num_batched_tokens: 16384
  max_model_len: 8192
  scheduler_delay_ms: 5
  request_timeout_ms: 30000
```

- `max_num_seqs` 太小：GPU 吃不满，吞吐低；
- `max_num_seqs` 太大：KV Cache 压力升高，TTFT 可能变差；
- `max_num_batched_tokens` 太小：长 prompt prefill 被切得太碎；
- `max_model_len` 盲目放大：单请求能力增强，但整体并发下降；
- `scheduler_delay_ms` 过大：更容易凑 batch，但交互体验变差。

调参不能只看 QPS，要同时看 P95 TTFT、P95 TPOT、OOM、拒绝率、GPU 利用率和业务质量。

### 5. Prefill 优化方法

Prefill 受输入长度影响很大，常见优化：

- 缩短系统 Prompt，移除重复工具说明；
- RAG 只注入必要证据，避免 top_k 过大；
- 对固定系统提示词、工具描述使用 Prefix Cache；
- 长上下文任务与普通聊天分池，避免互相拖慢；
- 对超长文档改成分段总结或异步任务，而不是强行塞入单次上下文。

```python
def choose_pool(input_tokens: int, task_type: str) -> str:
    if task_type in {"batch_summary", "long_document_qa"} or input_tokens > 12000:
        return "long_context_pool"
    return "interactive_pool"  # 中文注释：短交互和长上下文分池，避免长 prefill 拖慢在线问答
```

### 6. Decode 优化方法

Decode 主要影响完整生成耗时和输出吞吐，常见优化：

- 控制 `max_tokens`，避免无边界长输出；
- 选择合适的量化方案，平衡速度、显存和质量；
- 使用连续批处理提高 GPU 利用率；
- 对长输出任务评估投机解码；
- 根据模型和硬件选择 tensor parallel、pipeline parallel 或多副本；
- 流式返回并在客户端取消时停止后端生成。

注意：量化和投机解码都要用业务评测集验证，不能只看 tokens/s。

### 7. 投机解码适用场景

投机解码更适合：

- 输出较长、decode 占主要耗时的任务；
- 小模型和大模型分布接近，候选 token 接受率高；
- GPU/部署成本允许多跑一个 draft model；
- 对最终质量要求高，仍希望由大模型验收。

不适合：

- 输入很长但输出很短，瓶颈主要在 prefill；
- 任务高度依赖复杂推理，小模型候选命中率低；
- 结构化输出极严格且回退频繁；
- 服务资源紧张，多部署一个小模型反而抢占容量。

### 8. 推理服务的优先级调度

生产系统通常需要按任务类型和租户等级做调度：

| 请求类型 | 调度策略 |
| --- | --- |
| 在线短问答 | 高优先级、短队列、低 TTFT |
| 长文档总结 | 低优先级或异步队列 |
| 批量抽取 | 限速、可排队、可重试 |
| 高价值租户 | 保留配额或独立模型池 |
| 低风险简单任务 | 可切小模型或降低 max_tokens |

```python
def admit_request(priority: str, estimated_tokens: int, kv_cache_usage: float) -> str:
    if kv_cache_usage > 0.9 and priority != "high":
        return "reject_or_async"
    if estimated_tokens > 16000:
        return "route_to_long_context_pool"
    return "accept"  # 中文注释：准入策略要保护在线高优先级请求的 TTFT
```

### 9. 常见线上问题与处理

| 问题 | 表现 | 排查与处理 |
| --- | --- | --- |
| TTFT 突然升高 | 用户长时间无首字 | 看队列等待、prefill 长度、Prefix Cache 命中、GPU 是否排满 |
| TPOT 变慢 | 流式 token 间隔变大 | 看 batch 配置、GPU 利用率、decode 吞吐、是否混入长输出 |
| 并发高时 OOM | 模型进程重启 | 降低 max_num_seqs/max_model_len，限制长上下文，开启分页 KV |
| GPU 利用率低 | 成本高但吞吐低 | 检查 batch 是否太小、CPU 前处理/网络是否瓶颈 |
| 短请求被拖慢 | P95 长尾明显 | 长短请求分池，优先级队列，限制长文档同步请求 |

### 10. 面试中的设计回答框架

如果面试官问“如何优化一个私有化 LLM 推理服务”，可以按这个顺序回答：

```text
指标拆解 → 区分 prefill/decode → 观察队列与 KV Cache → 调整 batch 和上下文 → 使用缓存/投机解码/量化 → 分池和优先级 → 压测验证质量与成本
```

这比直接说“换 vLLM、加 GPU、做量化”更像工程方案。

## 高频面试问题与标准答案

**Q1：LLM 推理里的 prefill 和 decode 有什么区别？**

标准答案：我会把一次生成拆成 prefill 和 decode。Prefill 是处理输入 prompt，主要受输入 token 数影响，也决定首 token 前的等待；decode 是逐 token 生成，主要影响后续 token 的速度和完整耗时。所以 TTFT 高时我先看排队和 prefill，TPOT 高时我再看 decode 吞吐、batch 调度和 GPU 利用率。

**Q2：为什么模型能加载成功，但并发一上来就 OOM？**

标准答案：因为显存不只放模型权重，还要放 KV Cache 和运行时开销。KV Cache 会随并发数、上下文长度和输出长度增长。比如长文档 RAG 请求很多时，即使 7B 模型权重能放进显存，也可能因为 KV Cache 被打满而 OOM。处理上我会限制 `max_model_len` 和 `max_num_seqs`，做 token 级准入控制，长上下文任务单独分池，必要时用 PagedAttention 或增加副本。

**Q3：KV Cache 的作用是什么？**

标准答案：KV Cache 是缓存历史 token 在 attention 里的 Key 和 Value。没有它，每生成一个新 token 都要重复计算前面所有 token，decode 会非常慢。有了 KV Cache，生成下一个 token 时可以复用历史计算结果。但代价是显存占用随上下文和并发增长，所以面试中我会同时强调它提升速度，也会成为高并发长上下文的显存瓶颈。

**Q4：vLLM 的 PagedAttention 主要解决什么问题？**

标准答案：PagedAttention 主要解决 KV Cache 的显存管理问题。它把 KV Cache 按块管理，类似分页，减少动态请求场景下的显存碎片，让连续批处理和高并发更容易跑起来。它不是提升模型能力，而是提升推理服务的吞吐和显存利用率，但仍然要受总显存、上下文长度和并发数限制。

**Q5：连续批处理为什么能提升吞吐？有什么风险？**

标准答案：LLM 请求长度差异很大，如果用静态 batch，短请求会等长请求。连续批处理可以在每个 decode step 动态加入新请求、移除结束的请求，让 GPU 更持续地工作，所以吞吐更高。风险是如果为了凑 batch 引入过长调度等待，TTFT 会变差；并发过高也会推高 KV Cache 压力。因此我会同时观察吞吐、P95 TTFT、OOM 和拒绝率。

**Q6：Prefix Cache 和普通业务缓存有什么区别？**

标准答案：Prefix Cache 复用的是相同前缀 prompt 的模型中间状态，也就是 KV Cache，通常用于固定系统提示词、工具说明或固定模板；普通业务缓存复用的是最终答案。Prefix Cache 对 token 前缀一致性要求更高，但命中后可以减少 prefill 成本；业务缓存命中后甚至可以不调用模型，但要特别注意权限、版本和数据时效。

**Q7：投机解码是怎么加速的？会不会影响质量？**

标准答案：投机解码不是直接让小模型回答，而是让小模型先生成候选 token，再由大模型批量验证。被大模型接受的 token 可以一次推进多个生成步骤，从而降低平均 decode 时间。理论上最终由大模型验收，质量可以保持，但实际收益取决于小模型候选命中率和验证开销。如果小模型和大模型差异大，回退很多，可能加速不明显甚至更慢。

**Q8：什么场景适合投机解码？**

标准答案：我会优先考虑输出较长、decode 占主要耗时的场景，比如长报告生成、长摘要、批量文案生成。同时 draft model 要和目标模型分布接近，接受率才高。如果是输入很长但只输出很短的分类任务，瓶颈主要在 prefill，投机解码收益就有限；如果结构化输出非常严格，也要验证回退率和格式通过率。

**Q9：线上 TTFT 突然升高，你怎么排查？**

标准答案：我会先看 TTFT 是不是队列等待导致的，比如请求是否在模型网关或推理引擎前排队。然后看输入 token 分布有没有变长、Prefix Cache 命中率是否下降、长上下文请求是否挤占了普通请求。再看 GPU 利用率和 KV Cache 水位。如果是长请求拖慢短请求，我会做长短分池和优先级调度；如果是容量打满，就限流、扩副本或缩短上下文。

**Q10：TPOT 变慢和 TTFT 变慢的处理思路有什么不同？**

标准答案：TTFT 更关注“首 token 前发生了什么”，包括排队、prefill 和输入长度；TPOT 更关注“生成过程中每个 token 多慢”，包括 decode batch、GPU 利用率、量化、投机解码和输出长度。如果 TTFT 高，我会先减输入、查队列和缓存；如果 TPOT 高，我会看连续批处理配置、tokens/s、是否有长输出任务混在一起，以及是否需要调模型并行或投机解码。

**Q11：如何给推理服务做优先级调度？**

标准答案：我会按租户等级、任务类型和 token 预算分层。在线短问答走高优先级短队列，长文档总结和批量抽取走低优先级或异步队列；高价值租户保留配额；当 KV Cache 水位过高时，低优先级长请求可以排队、转异步或拒绝，不能让它们拖垮所有用户的 TTFT。

**Q12：优化推理服务后怎么证明没有伤害业务效果？**

标准答案：我不会只看 tokens/s。推理优化后要同时看性能和质量：性能看 TTFT、TPOT、P95/P99、吞吐、OOM、拒绝率和 GPU 利用率；质量看固定评测集的准确率、结构化输出成功率、工具调用参数正确率和人工抽检。像量化、投机解码、上下文裁剪这类优化尤其要做灰度和回放，确认速度提升没有换来质量下降。
