# LLM 线上问题排查与 Trace 回放

## 主题选择记录

- **本次序号**：第 42 篇。
- **README 位置**：追加到目录表末尾，归入「生产化与 LLMOps」卷。
- **选题理由**：README 已有「LLM 应用评估与可观测性」讲指标与监控，「LLM 数据闭环与 Bad-Case 归因」讲问题样本沉淀，「AI Agent 评估与轨迹分析」讲 Agent 过程评分。但面试中还经常单独追问：线上用户反馈“答错了”之后，工程师如何从 `trace_id` 复现链路、定位根因、验证修复并防止回归。本篇聚焦 LLM 应用的线上排障方法论与 Trace 回放设计。
- **避免重复**：不重复展开通用评测体系、Agent 轨迹评分、RAG 指标或数据闭环管理，只聚焦一次线上请求从日志采集、链路拼接、回放复现、根因定位到修复验证的工程闭环。

## 核心概念

### 1. Trace 是一次 LLM 请求的事实链

Trace 不是简单日志，而是一次用户请求从入口到输出的完整事实链。它至少要回答四个问题：用户问了什么，系统实际喂给模型什么，中间调用了哪些检索或工具，最终为什么给出这个答案。

在 LLM 应用里，Trace 通常包含：

| 字段 | 说明 |
| --- | --- |
| trace_id / request_id | 串联网关、RAG、模型、工具、输出和反馈 |
| user / tenant / session | 脱敏后的用户、租户、会话与轮次 |
| versions | Prompt、模型、检索索引、工具 Schema、安全策略版本 |
| retrieval | 改写 query、召回文档 ID、分数、重排结果、引用片段 |
| llm_call | 模型别名、参数、输入摘要、输出摘要、token、延迟 |
| tool_call | 工具名、参数摘要、返回摘要、错误码、重试次数 |
| final_output | 最终答案、引用、结构化字段、拒答原因 |
| feedback | 点赞点踩、人工审核、业务结果、用户追问 |

面试表达要强调：**Trace 的价值不是多打日志，而是让线上问题可复现、可定位、可审计、可回归。**

### 2. 线上排障的核心是“先复现，再归因”

LLM 应用出错后，不能一上来就改 Prompt。正确顺序是先拿到 `trace_id`，还原当时的输入、版本、检索结果、工具返回和安全策略，再判断错误发生在哪一层。

常见错误层次：

1. **输入层**：用户问题歧义、多轮上下文缺失、意图识别错。
2. **检索层**：query 改写错、召回漏、重排错、ACL 过滤错。
3. **上下文层**：正确片段没放进 Prompt、上下文被截断、引用顺序干扰。
4. **模型层**：模型没有遵循证据、格式错、幻觉、拒答策略不当。
5. **工具层**：工具选错、参数错、超时、返回被模型误读。
6. **策略层**：安全拦截、租户配置、灰度版本、实验分流导致行为不同。

面试中可以把这套逻辑概括为：**先看输入和版本，再看检索和工具，最后看模型输出；不要把所有问题都归因成模型不稳定。**

### 3. Trace 回放不是重放线上副作用

回放的目标是复现“当时模型看到什么、系统做了什么判断”，不等于再次调用生产工具执行写操作。尤其是退款、下单、发邮件、改权限这类高风险工具，回放环境必须使用快照、Mock、沙箱或 dry-run。

一次可靠回放通常需要固定：

- Prompt 模板与变量渲染结果。
- 模型别名与实际模型版本。
- 生成参数，如 temperature、top_p、max_tokens。
- 检索索引版本、召回结果快照和文档权限版本。
- 工具 Schema、工具返回快照和错误码。
- 安全策略、租户配置与实验分组。

如果只保存最终问答，后续就很难判断问题到底是知识缺失、检索漏召回、模型幻觉还是工具异常。

### 4. Trace 要兼顾排障与隐私

企业面试里很容易追问：日志打这么细，会不会泄露用户数据？答案是 Trace 设计必须从一开始就做数据治理：

- 长期日志只存必要摘要、ID、哈希和脱敏字段。
- 原始 Prompt、工具返回和用户输入进入受控存储，设置访问权限与保留周期。
- 对身份证、手机号、地址、合同金额等敏感字段做脱敏或分级存储。
- 高风险排障操作需要审计：谁在什么时间查看了哪条 Trace。
- 回放数据进入评测集前必须再次脱敏，避免把生产 PII 带入训练或评测。

## 核心知识点

### 1. Trace 采集的分层设计

推荐按 Span 记录关键步骤，而不是把所有内容塞进一条日志：

```text
http_request
  ├── intent_route
  ├── query_rewrite
  ├── retrieve
  ├── rerank
  ├── prompt_render
  ├── llm_generate
  ├── tool_call
  ├── output_validate
  └── feedback
```

每个 Span 至少记录：开始时间、结束时间、状态、版本、输入摘要、输出摘要、错误码和上游 `trace_id`。这样排查 P95 延迟、检索漏召回、工具超时和模型格式失败时，都能定位到具体环节。

### 2. 最小可用 Trace 字段

面试中可以给出一套“够用但不过度”的字段清单：

```json
{
  "trace_id": "tr_20260612_001",
  "tenant_id": "t_001",
  "session_id": "s_7788",
  "turn_id": 3,
  "scenario": "customer_support_rag",
  "versions": {
    "prompt": "support_rag_v12",
    "model": "gpt-4.1-mini@2026-05",
    "index": "kb_20260610",
    "reranker": "bge-reranker-v2",
    "policy": "acl_policy_v5"
  },
  "retrieval": {
    "query": "退款多久到账",
    "doc_ids": ["refund_policy_03", "sla_02"],
    "scores": [0.83, 0.79]
  },
  "llm": {
    "temperature": 0.2,
    "input_tokens": 1830,
    "output_tokens": 220,
    "latency_ms": 1460
  },
  "result": {
    "answer_hash": "sha256:...",
    "citation_ids": ["refund_policy_03"],
    "validated": true
  }
}
```

生产里不一定长期保存完整 Prompt，但至少要保存可追溯的模板版本、变量摘要、引用 ID、模型参数和输出哈希。高风险场景可以把完整输入输出加密存储，并设置更短保留周期。

### 3. 排障路径：从现象到根因

可以用一个固定排障表来回答面试官：

| 现象 | 优先检查 | 常见根因 | 修复方向 |
| --- | --- | --- | --- |
| 答案事实错误 | 召回文档、引用、Prompt 中证据 | 知识过期、召回漏、模型幻觉 | 修知识、调检索、强化忠实约束 |
| 有答案却拒答 | 阈值、安全策略、上下文截断 | 阈值过高、证据没进上下文 | 调阈值、改 Prompt、补评测样本 |
| 引用不对 | citation_id、片段映射、重排结果 | 引用与答案不同源 | 引用校验、答案句子级归因 |
| 格式错误 | 输出 Schema、解析错误、重试日志 | Schema 太复杂、示例不足 | 简化 Schema、校验修复、有限重试 |
| 延迟升高 | 各 Span 耗时、token、工具重试 | 上下文过长、检索慢、模型队列 | 压缩上下文、缓存、降级路由 |
| 成本异常 | token 分布、模型路由、循环次数 | 命中大模型、重复工具调用 | 路由小模型、调用预算、停止条件 |
| 越权风险 | ACL 版本、租户、缓存 key | 权限过滤漏、缓存串租户 | 权限前置、缓存隔离、审计告警 |

核心思路是：**先定位层，再选择修复手段**。如果根因是检索漏召回，单纯改 Prompt 只是掩盖问题；如果根因是工具返回错误，换模型也没有意义。

### 4. 回放系统的基本流程

一次 Trace 回放可以设计成下面的流程：

```text
选择 trace_id
  → 拉取脱敏 Trace 和版本信息
  → 加载 Prompt / 检索 / 工具快照
  → 在沙箱或 Mock 环境执行
  → 对比原输出、当前输出、期望输出
  → 标注根因与修复建议
  → 进入回归集或坏 Case 池
```

回放时要区分三种模式：

1. **只读复盘**：展示当时链路，不重新调用模型，适合客服和合规审计。
2. **确定性回放**：固定上下文、工具返回和模型参数，对比版本差异。
3. **修复验证回放**：替换新 Prompt、新索引或新策略，验证同类问题是否改善。

### 5. Python 示例：构造可脱敏的 Trace 事件

```python
from dataclasses import dataclass, asdict
from time import time
from typing import Any

SENSITIVE_KEYS = {"phone", "id_card", "address", "email"}

@dataclass
class TraceEvent:
    trace_id: str
    span: str
    status: str
    started_at: float
    ended_at: float
    version: str
    input_summary: dict[str, Any]
    output_summary: dict[str, Any]

def mask_sensitive(data: dict[str, Any]) -> dict[str, Any]:
    # 中文注释：长期日志只保留脱敏摘要，原文进入受控存储
    masked = {}
    for key, value in data.items():
        if key in SENSITIVE_KEYS and value:
            masked[key] = "***"
        else:
            masked[key] = value
    return masked

def emit_trace_event(
    trace_id: str,
    span: str,
    version: str,
    input_data: dict[str, Any],
    output_data: dict[str, Any],
    sink,
) -> None:
    started = time()
    event = TraceEvent(
        trace_id=trace_id,
        span=span,
        status="ok",
        started_at=started,
        ended_at=time(),
        version=version,
        input_summary=mask_sensitive(input_data),
        output_summary=mask_sensitive(output_data),
    )
    sink.write(asdict(event))
```

这段代码想表达的不是“生产就这么写”，而是面试中要讲清楚：Trace 事件要结构化、带版本、可串联、默认脱敏。

### 6. Python 示例：判断回放是否允许执行工具

```python
WRITE_TOOLS = {"create_refund", "send_email", "update_permission"}

def can_replay_tool(tool_name: str, replay_mode: str) -> bool:
    # 中文注释：回放环境禁止真实执行有副作用的工具
    if replay_mode == "production":
        return False
    if tool_name in WRITE_TOOLS:
        return replay_mode in {"mock", "sandbox", "dry_run"}
    return replay_mode in {"mock", "sandbox", "dry_run", "readonly"}

def replay_tool_call(tool_call: dict, replay_mode: str, mock_store: dict):
    tool_name = tool_call["name"]
    if not can_replay_tool(tool_name, replay_mode):
        raise PermissionError(f"tool {tool_name} is not allowed in replay")

    cache_key = tool_call["trace_tool_call_id"]
    if replay_mode in {"mock", "readonly"}:
        return mock_store[cache_key]

    return run_in_sandbox(tool_call)
```

面试时可以补一句：写操作工具必须支持幂等键、dry-run 或沙箱执行，否则 Trace 回放会变成新的生产风险。

### 7. Trace 与评测集、发布流程的关系

Trace 排障不是孤立工具，应该接入 LLMOps 流程：

- 线上反馈或告警触发 Trace 排查。
- 有代表性的坏 Trace 脱敏后进入回归评测集。
- 修复方案在同一批 Trace 上回放验证。
- 发布前跑回归，发布后监控同类指标是否下降。
- 如果新版本引入新错误，依靠版本字段快速回滚。

一句话总结：**Trace 负责复现事实，评测集负责防止回退，发布治理负责控制风险。**

## 高频面试问题与标准答案

**Q1：线上用户说 LLM 答错了，你怎么排查？**  
A：我不会先改 Prompt，而是先拿 `trace_id` 拉全链路。第一看用户原始输入、多轮上下文和意图路由是否正确；第二看 RAG 的 query 改写、召回文档、重排和引用是否包含正确答案；第三看 Prompt 渲染后是否把证据放进上下文；第四看模型输出有没有无视证据或格式失败；如果有工具调用，还要看工具参数、返回和重试。定位到具体层后再修，比如知识缺失就修知识库，召回漏就调检索，证据有但模型乱答才改 Prompt 或模型，并把这个 case 加入回归集。

**Q2：Trace 里必须记录哪些字段？**  
A：最少要有 `trace_id`、用户和租户的脱敏标识、session/turn、Prompt 版本、模型版本、生成参数、索引版本、检索 query 和召回文档 ID、工具调用摘要、最终输出、token、延迟、错误码和用户反馈。高风险场景还要记录权限策略版本和实验分组。核心不是把所有原文永久保存，而是保证能复盘当时系统看到什么、执行了什么、为什么这么回答。

**Q3：为什么只保存用户问题和最终答案不够？**  
A：因为最终问答只能说明“表面结果”，不能定位根因。比如答错可能是知识库没有这条政策，也可能是检索没召回，也可能是召回了但上下文被截断，还可能是工具返回错误或模型幻觉。如果没有中间链路，后续只能凭猜测改 Prompt，容易修错方向，也没法证明修复真的有效。

**Q4：怎么做 Trace 回放？**  
A：我会把回放做成沙箱流程：根据 `trace_id` 拉取脱敏 Trace、Prompt 版本、模型参数、检索结果快照、工具返回快照和策略版本，然后在 Mock 或 dry-run 环境复现原链路。回放结果要和原输出、期望输出对比，看新 Prompt、新索引或新策略是否修复问题。对于写操作工具，绝不能在回放里真实执行，只能用快照、Mock 或沙箱。

**Q5：线上复现不了原来的错误怎么办？**  
A：我会先检查版本是否一致，包括 Prompt、模型别名背后的实际模型、检索索引、工具 Schema、权限策略和实验分组；再看外部数据是否变化，比如知识库更新、工具接口返回变了。如果这些没有快照，就说明可复现性建设不足。短期可以用当时日志尽量还原，长期要补齐版本记录和关键输入输出快照。

**Q6：Trace 日志怎么处理隐私和合规？**  
A：原则是最小必要、默认脱敏、分级存储和可审计访问。长期日志存摘要、ID、哈希、版本和指标；原始输入输出如果必须保存，要加密、限权、设置保留周期。进入评测集或训练集前要再次脱敏，并记录谁查看过高敏 Trace。面试里我会强调：可观测性不能以泄露生产数据为代价。

**Q7：如果 RAG 答案幻觉，你怎么通过 Trace 定位？**  
A：先看召回文档里有没有正确答案。如果没有，就是知识缺失、query 改写、向量召回、混合检索或重排问题；如果有，再看正确片段是否进入最终 Prompt，有没有被截断；如果 Prompt 里有证据但模型仍编造，就是生成约束、引用校验或拒答策略问题。这样分层排查比直接说“模型幻觉”更工程化。

**Q8：Trace 和监控指标有什么区别？**  
A：指标告诉我“哪里异常”，Trace 告诉我“这次请求具体发生了什么”。比如监控发现 P95 延迟升高，只能知道慢了；Trace 能拆开看是检索慢、模型排队、上下文太长还是工具重试。两者关系是指标发现问题，Trace 定位原因，坏 Case 回流验证修复。

**Q9：如何防止 Trace 回放造成新的线上风险？**  
A：回放环境要和生产执行隔离，默认禁止真实写操作。只读工具可以用快照或 Mock，写操作必须走 sandbox、dry-run 或直接读取历史返回；同时工具层要校验幂等键和权限，回放账号不能有生产写权限。这样即使回放一条退款 Agent 的 Trace，也不会真的再次退款。

**Q10：你会怎么向面试官总结这套能力？**  
A：我会说 LLM 线上排障要做到“四个可”：可追踪、可复现、可归因、可回归。可追踪靠 `trace_id` 串起 Prompt、RAG、模型、工具和反馈；可复现靠版本、快照和沙箱回放；可归因靠分层判断输入、检索、上下文、模型、工具和策略；可回归靠把高价值坏 Trace 脱敏后加入评测集。这样才能从“用户说答错了”走到“我们知道错在哪里、怎么修、怎么证明不会再犯”。
