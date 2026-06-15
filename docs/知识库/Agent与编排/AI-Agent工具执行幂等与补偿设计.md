# AI Agent 工具执行幂等与补偿设计

## 主题选择记录

- **本次序号**：第 43 篇。
- **README 位置**：追加到目录表末尾，归入「Agent 与编排」卷。
- **选题理由**：README 已有 Agent 工具调用、规划执行、多 Agent、工具沙箱、Agent 评估，以及会话状态与检查点恢复，但面试里还经常追问：**Agent 调用了写工具以后超时怎么办？如何避免重复下单、重复退款、重复发通知？工具执行一半失败怎么补偿？重试、幂等、对账和人工确认的边界是什么？** 本篇专门展开 Agent 工具执行层的可靠性设计。
- **避免重复**：不重复讲 Function Calling 的参数生成，不重复讲通用异步任务架构中的任务队列，也不重复第 42 篇的整体 checkpoint 恢复；本文只聚焦 **工具执行的副作用控制、幂等键、执行记录、对账、重试、补偿和高风险动作确认**。

## 核心概念

### 1. Agent 工具执行为什么比普通函数调用更危险

普通函数调用通常由确定性代码触发，调用路径相对固定；Agent 工具调用则由模型根据上下文动态选择，风险更高：

- 模型可能误判用户意图，把“查询退款”理解成“发起退款”。
- 参数格式合法，但业务语义错误，例如订单号不属于当前用户。
- 工具超时后模型或 Worker 重试，导致重复创建工单、重复发送消息。
- 多步任务中前一步已经产生副作用，后一步失败，系统进入部分成功状态。
- 用户、模型、工具、队列和外部系统之间存在网络不确定性，响应丢失不等于执行失败。

面试中要先定调：**Agent 可以重新规划、重新生成参数，但副作用工具不能随便重复执行**。只靠 Prompt 说“不要重复调用”是不够的，必须在工具执行层做系统级约束。

### 2. 副作用工具是什么

副作用工具是指会改变外部系统状态、产生费用、影响用户权益或触发真实动作的工具。常见分类如下：

| 工具类型 | 示例 | 风险 |
| --- | --- | --- |
| 只读工具 | 查询订单、检索知识库、查库存 | 数据过期、越权读取、重试成本 |
| 低风险写工具 | 创建草稿、记录日志、保存偏好 | 重复数据、脏状态 |
| 中风险写工具 | 创建工单、发送邮件、推送通知 | 重复通知、重复工单、用户打扰 |
| 高风险写工具 | 退款、转账、改权限、删除数据 | 资金损失、权限事故、合规风险 |
| 长事务工具 | 批量导入、文档解析、报表生成 | 部分完成、任务悬挂、资源浪费 |

面试回答时建议明确：**只读工具可以有限重试，写工具必须幂等，高风险写工具还要人工确认和审计，长事务工具要有任务状态和补偿策略**。

### 3. 幂等、重试、对账、补偿的区别

这四个词经常一起出现，但职责不同：

- **重试**：处理临时失败，比如网络抖动、429、部分 5xx。它解决“这次没成功，能不能再试一次”。
- **幂等**：同一个业务请求执行多次，结果只生效一次。它解决“重复请求会不会造成重复副作用”。
- **对账**：响应不确定时，查询外部系统或执行记录，确认动作到底有没有发生。它解决“不知道成功还是失败”的灰区。
- **补偿**：部分步骤已成功、后续步骤失败时，用反向动作或人工流程修正状态。它解决“不能原子回滚时如何恢复业务一致性”。

一句面试表达：**重试提高成功率，幂等控制重复副作用，对账消除未知状态，补偿处理部分成功后的业务修复**。

### 4. 幂等不是简单去重

很多人会把幂等理解成“请求重复就丢弃”，这是不完整的。真正的幂等要绑定业务语义：

- 同一个用户、同一个订单、同一个动作、同一次 Agent run，应该生成同一个幂等键。
- 参数不同但幂等键相同，必须拒绝或进入人工检查，不能静默复用旧结果。
- 幂等记录要保存执行状态：`pending`、`running`、`succeeded`、`failed`、`unknown`。
- 幂等结果要可查询，重试时返回同一业务结果，例如同一个工单 ID。

因此幂等不是“防止 API 被调用两次”这么简单，而是让系统在重复、超时、重启、消息重复投递时仍能保持业务结果一致。

## 核心知识点

### 1. 工具风险分级与执行策略

Agent 工具执行层应先给每个工具声明风险等级，而不是让模型自行判断。

| 风险等级 | 工具示例 | 执行策略 |
| --- | --- | --- |
| read_only | `get_order_status`、`search_docs` | 可重试，必须做权限校验和超时控制 |
| safe_write | `save_draft`、`create_note` | 幂等键 + 执行记录，失败可重试 |
| side_effect | `create_ticket`、`send_email` | 幂等键 + 对账 + 重试上限 + 告警 |
| critical | `refund_order`、`grant_role` | 人工确认 + 审批记录 + 强审计 + 补偿预案 |

示例工具元数据：

```python
TOOLS = {
    "get_order_status": {
        "risk": "read_only",
        "timeout_ms": 1000,
        "retryable": True,
    },
    "create_ticket": {
        "risk": "side_effect",
        "timeout_ms": 3000,
        "retryable": True,
        "requires_idempotency_key": True,
    },
    "refund_order": {
        "risk": "critical",
        "timeout_ms": 5000,
        "retryable": False,
        "requires_human_approval": True,
        "requires_idempotency_key": True,
    },
}
```

面试中可以补一句：工具描述给模型看只是第一层，真正的风险策略要在执行器里强制执行。模型即使生成了高风险工具调用，也必须过权限、审批和幂等检查。

### 2. 幂等键如何设计

幂等键要表达“同一个业务动作”，常见组成：

```text
tenant_id:user_id:business_object:action:run_id_or_request_id
```

例如：

```text
t_001:u_123:order_A1001:create_ticket:run_7788
t_001:u_123:order_A1001:refund:approval_5566
```

设计要点：

1. **不要只用随机 UUID**：随机 UUID 每次都不同，无法识别业务重复。
2. **不要只用用户输入文本**：同义改写会导致不同 key，且可能包含敏感信息。
3. **要包含租户和用户边界**：避免跨租户或跨用户误复用。
4. **要包含业务对象和动作**：查询订单和创建退款不能共用 key。
5. **高风险动作绑定审批 ID**：没有审批就不能构造可执行幂等键。

简化实现：

```python
import hashlib
import json


def build_idempotency_key(
    tenant_id: str,
    user_id: str,
    action: str,
    business_args: dict,
    run_id: str,
) -> str:
    # 中文注释：只对稳定业务字段做哈希，避免把敏感原文直接放进幂等键
    stable_payload = {
        "tenant_id": tenant_id,
        "user_id": user_id,
        "action": action,
        "business_args": business_args,
        "run_id": run_id,
    }
    raw = json.dumps(stable_payload, sort_keys=True, ensure_ascii=False)
    digest = hashlib.sha256(raw.encode("utf-8")).hexdigest()[:24]
    return f"{tenant_id}:{user_id}:{action}:{digest}"
```

面试高频追问是“幂等键是否应该包含 run_id”。回答要看业务：如果同一次 Agent run 的重复执行必须合并，可以包含 run_id；如果用户刷新页面后同一个业务动作也要合并，则应使用客户端 request_id、审批 ID 或业务对象版本，而不是新的 run_id。

### 3. 工具执行记录表

幂等需要落库，不能只放内存。一个简化的执行记录表：

```sql
CREATE TABLE agent_tool_executions (
    idempotency_key TEXT PRIMARY KEY,
    tenant_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    run_id TEXT NOT NULL,
    tool_name TEXT NOT NULL,
    args_hash TEXT NOT NULL,
    status TEXT NOT NULL,
    external_request_id TEXT,
    result_json JSON,
    error_code TEXT,
    attempt_count INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL
);
```

关键字段说明：

| 字段 | 面试关注点 |
| --- | --- |
| `idempotency_key` | 重复执行时的稳定查找键 |
| `args_hash` | 防止同一 key 搭配不同参数导致脏复用 |
| `status` | 区分成功、失败、执行中、未知 |
| `external_request_id` | 对账外部系统的凭证 |
| `attempt_count` | 控制重试上限，避免无限重试 |

注意：`status=unknown` 很重要。很多故障不是明确失败，而是“调用发出去了，但响应没回来”。这时不能直接重试写工具，要先对账。

### 4. 工具执行器的核心流程

一个可靠的执行器通常按以下顺序处理：

```text
权限校验
  → 风险等级检查
  → 只读工具走安全重试路径
  → 人工确认检查
  → 生成或校验幂等键
  → 原子创建 running 占位记录
  → 如果唯一键冲突，读取已有执行记录并校验参数 hash
  → 调用外部工具
  → 保存结果或 unknown
  → 返回结构化 observation
```

这里的关键不是“先查再插”，而是依赖数据库唯一约束做 **原子 insert / upsert / 行锁**。否则两个 Worker 并发执行时可能同时查不到记录，随后都调用外部写工具。面试里要主动说明：幂等表的 `idempotency_key` 必须有唯一约束，冲突后读取已有记录并按状态返回、对账或转人工。

简化代码：

```python
def execute_tool(tool_name: str, args: dict, context: dict) -> dict:
    policy = TOOLS[tool_name]
    check_permission(context["user"], tool_name, args)

    if policy["risk"] == "read_only":
        # 中文注释：只读工具没有真实副作用，可走受控重试，但仍要做权限、超时和限流
        return retry_read_tool(tool_name, args, timeout_ms=policy["timeout_ms"])

    if policy.get("requires_human_approval"):
        approval_id = args.get("approval_id")
        if not approval_service.is_approved(approval_id, context["user"]):
            return {"ok": False, "error": "human_approval_required"}

    idempotency_key = args.get("idempotency_key")
    if policy.get("requires_idempotency_key") and not idempotency_key:
        return {"ok": False, "error": "idempotency_key_required"}

    args_hash = hash_args(args)
    try:
        # 中文注释：依赖唯一约束原子创建占位记录，避免并发 Worker 同时执行写工具
        execution_repo.insert_running(idempotency_key, tool_name, args_hash, context)
    except DuplicateKeyError:
        existing = execution_repo.find_for_update(idempotency_key)
        if existing.args_hash != args_hash:
            return {"ok": False, "error": "idempotency_key_conflict"}
        if existing.status == "succeeded":
            return existing.result_json
        if existing.status == "unknown":
            # 中文注释：响应未知时先对账，不能直接重复执行副作用工具
            return reconcile_tool_execution(existing)
        return {"ok": False, "error": "execution_in_progress"}

    try:
        result = call_external_tool(tool_name, args, timeout_ms=policy["timeout_ms"])
        execution_repo.mark_succeeded(idempotency_key, result)
        return result
    except TimeoutError:
        execution_repo.mark_unknown(idempotency_key, error_code="timeout")
        return {"ok": False, "error": "execution_unknown", "retry_after_reconcile": True}
    except RetryableToolError as exc:
        execution_repo.mark_failed(idempotency_key, error_code=exc.code)
        return retry_if_allowed(tool_name, args, context, exc)
```

这段逻辑的面试价值在于：它没有把模型当成可靠边界，而是在工具执行器里保证权限、审批、幂等、冲突检测和未知状态处理。

### 5. 响应未知时如何对账

外部工具超时后，最危险的判断是“超时等于失败”。实际上可能有三种情况：

1. 请求没有到达外部系统。
2. 请求到达了，但外部系统执行失败。
3. 请求执行成功，但响应在网络中丢失。

因此副作用工具超时后要进入对账流程：

```python
def reconcile_tool_execution(record: dict) -> dict:
    if record["external_request_id"]:
        external = external_api.query_by_request_id(record["external_request_id"])
    else:
        external = external_api.query_by_idempotency_key(record["idempotency_key"])

    # 中文注释：对账成功后补齐本地执行记录，后续重试直接返回同一结果
    if external["status"] == "succeeded":
        result = {"ok": True, "data": external["data"]}
        execution_repo.mark_succeeded(record["idempotency_key"], result)
        return result

    if external["status"] == "not_found":
        execution_repo.mark_failed(record["idempotency_key"], "not_executed")
        return {"ok": False, "error": "safe_to_retry"}

    return {"ok": False, "error": "still_unknown", "needs_human_review": True}
```

如果外部系统不支持按幂等键或 request_id 查询，就要降低自动化级别：应用层执行表、业务锁、人工复核、或把工具改造成异步任务并返回 `job_id`。

### 6. 重试策略：不是所有失败都能重试

可以重试的失败：

- 网络抖动、连接重置。
- 429 限流，且有退避策略。
- 部分 5xx，外部系统明确未执行或接口幂等。
- 只读工具超时。

不应该直接重试的失败：

- 权限不足、租户不匹配。
- 参数校验失败、业务规则不允许。
- 高风险写工具响应未知。
- 外部系统返回“处理中”或“状态不明”。
- 已经超过最大尝试次数。

推荐回答方式：**先分类，再决定重试；读工具和幂等写工具可以退避重试，高风险或未知状态要先对账或转人工**。

### 7. 补偿设计：无法强事务时如何保证业务一致性

Agent 常会跨多个系统调用工具，例如：

```text
创建退款单 → 发送用户通知 → 更新 CRM 备注
```

如果退款单创建成功，但通知失败，不能简单数据库回滚，因为外部系统副作用已经发生。常见补偿方式：

| 场景 | 补偿策略 |
| --- | --- |
| 工单重复创建 | 合并工单、关闭重复工单、保留审计记录 |
| 通知发送失败 | 进入待重发队列，超过上限转人工 |
| 权限修改后后续失败 | 执行反向权限变更，并记录审批与回滚原因 |
| 退款成功但 CRM 更新失败 | 以退款系统为准，CRM 标记待同步 |
| 多系统状态不一致 | 通过对账任务定期修复，严重情况人工处理 |

面试中不要承诺“所有操作都能自动回滚”。更真实的表达是：外部系统通常没有分布式事务，应该用 Saga 思路，把每一步做成可追踪状态，并为关键步骤设计补偿动作或人工处理路径。

### 8. 高风险工具的人机协同

高风险工具不应由 Agent 自主最终提交。推荐流程：

```text
Agent 生成操作建议
  → 系统校验权限、金额、对象、风险策略
  → 展示给用户或审批人确认
  → 确认后生成 approval_id
  → 工具执行器使用 approval_id + idempotency_key 执行
  → 保存审计记录
```

对于退款、转账、删除、权限变更这类工具，面试里要强调：

- Agent 可以准备草稿和解释理由。
- 最终执行需要人类确认或业务审批。
- 审批内容要绑定关键参数，确认后参数不能被模型偷偷改掉。
- 执行后要记录审批人、时间、参数 hash、外部 request_id 和结果。

### 9. 和 checkpoint 的配合

第 42 篇讲的是 Agent 运行状态和检查点，本篇强调工具执行记录。两者关系是：

- **checkpoint** 记录 Agent 当前走到哪一步、下一步准备做什么。
- **tool execution record** 记录某个工具动作是否已经发生、结果是什么。

恢复时要先读 checkpoint，再查工具执行记录：

```text
checkpoint 显示下一步是 create_ticket
  → 根据 idempotency_key 查 execution record
  → succeeded：把同一个 ticket_id 写回 state，继续后续步骤
  → unknown：先对账
  → failed 且可重试：按策略重试
  → conflict：停止并转人工
```

这样才能避免“Agent 状态恢复了，但工具副作用重复发生”。

### 10. 常见设计误区

1. **超时后直接重试写工具**：可能重复扣款、重复发通知。
2. **只在 Prompt 里要求不要重复调用**：模型约束不是系统边界。
3. **幂等键用随机 UUID**：每次请求都不同，无法识别业务重复。
4. **不保存参数 hash**：同一幂等键可能被不同参数复用，造成数据污染。
5. **没有 unknown 状态**：把未知当失败处理，容易重复执行。
6. **高风险动作无人工确认**：面试中会被追问安全和审计。
7. **补偿等同于回滚**：外部系统通常不能原子回滚，要按业务设计补偿。

## 高频面试问题与标准答案

### 问题 1：Agent 调用工具时，为什么要特别强调幂等？

标准答案：因为 Agent 工具调用不是普通的一次函数调用，它可能因为模型重规划、Worker 重启、队列重复投递、接口超时而重复执行。对于只读工具，重复最多增加成本；但对于创建工单、发通知、退款、改权限这类写工具，重复执行会产生真实副作用。

我的做法是在工具执行层强制幂等，而不是只靠 Prompt。每个副作用工具都要有业务幂等键、执行记录、参数 hash 和状态。重复请求进来时，如果已经成功就返回同一个结果；如果状态未知就先对账；如果同一个 key 搭配了不同参数，就拒绝并转人工。

### 问题 2：幂等键一般怎么设计？

标准答案：幂等键要绑定业务语义，通常包含租户、用户、业务对象、动作和一次稳定请求标识。比如 `tenant:user:order:refund:approval_id`，而不是每次生成一个随机 UUID。随机 UUID 只能标识一次请求，不能识别“同一个业务动作被重复提交”。

我还会保存参数 hash。因为同一个幂等键如果传了不同金额或不同订单号，不能直接复用旧结果，应该返回冲突错误或者进入人工处理。高风险动作还会把幂等键和审批 ID 绑定，确保用户确认的参数和真正执行的参数一致。

### 问题 3：工具调用超时了，你会不会重试？

标准答案：不会一概而论。我会先看工具类型和失败状态。只读工具超时可以有限重试；幂等写工具可以在确认安全的情况下退避重试；但副作用工具如果请求已经发出，只是响应超时，就不能把它当成失败直接再调一次。

更稳妥的做法是把本地执行记录标记成 `unknown`，然后用 external_request_id 或 idempotency_key 去外部系统对账。确认没有执行过，才可以重试；确认已执行成功，就补齐本地记录并返回同一个结果；如果仍然不确定，就转人工或进入异常处理队列。

### 问题 4：重试、幂等、补偿分别解决什么问题？

标准答案：重试解决临时失败，提高调用成功率；幂等解决重复请求导致重复副作用的问题；补偿解决多步骤流程中部分步骤已经成功、后续步骤失败的问题。它们不是一回事。

比如创建工单接口 500，可以重试；如果请求超时但可能已经创建成功，就要靠幂等键避免重复创建；如果工单创建成功但通知失败，就要用补偿策略，比如进入待通知队列或人工处理，而不是简单说整体回滚。面试里我会强调：先做失败分类，再选择重试、对账、补偿或人工介入。

### 问题 5：如果外部系统不支持幂等接口怎么办？

标准答案：首先我会尽量在接入层补一个应用侧幂等层，比如本地执行记录表、业务锁、参数 hash、状态机和对账任务。请求进入时先查本地执行记录，确保同一个业务动作不会被并发执行多次。

但如果外部系统既不支持幂等，也不支持按 request_id 查询结果，那自动化级别就要降低。对于低风险动作可以做保守重试和人工排查；对于高风险动作，比如退款、转账、改权限，我会要求人工确认，并在响应未知时停止自动重试，转人工对账。不能为了 Agent 自动化牺牲业务安全。

### 问题 6：如何避免 Agent 重复创建工单？

标准答案：我会把创建工单定义为副作用工具，要求必须传幂等键。幂等键可以由租户、用户、问题类型、业务对象和 run_id 或 request_id 组成。执行前先查 `agent_tool_executions` 表，如果同一个 key 已经成功，就直接返回原来的 ticket_id；如果正在执行，就返回处理中；如果状态未知，就先到工单系统按 request_id 或幂等键查询。

另外，还要校验参数 hash，避免同一个幂等键被不同问题内容复用。对于用户连续多次表达同一问题，也可以在业务层做相似工单合并，但那是业务去重；工具幂等解决的是同一次动作重复执行的问题。

### 问题 7：Agent 执行多步工具，其中一步成功、后一步失败怎么办？

标准答案：这类场景不能简单当成事务回滚，因为很多外部系统已经产生了不可撤销副作用。我会用 Saga 的思路，把每一步记录成状态，并为关键步骤设计补偿动作。

举个例子，退款单创建成功，但发送用户通知失败。退款本身应该以退款系统状态为准，不能为了通知失败去撤销退款；补偿方式是把通知放入待重发队列，超过上限后转人工。再比如权限变更后后续流程失败，如果业务允许，就执行反向权限变更，并记录审批和回滚原因。关键是承认外部系统没有统一事务，用状态、对账和补偿来保证最终一致。

### 问题 8：高风险工具为什么还需要人工确认？有了权限校验不够吗？

标准答案：权限校验只能说明“这个用户有没有资格做这件事”，不能说明“这次具体操作是否经过明确确认”。Agent 可能误解用户意图，也可能生成了格式正确但业务上危险的参数。退款、转账、删除数据、改权限这类动作，一旦执行就可能造成资金或合规问题，所以需要人工确认或审批。

我的设计是 Agent 只生成操作建议和参数草稿，系统侧校验后展示给用户或审批人。确认后生成 approval_id，并把审批 ID、参数 hash、幂等键绑定。真正执行时如果参数变了，就拒绝执行。这样既利用 Agent 提效，又把最终决策权留在确定性的审批流程里。

### 问题 9：工具执行记录和 Agent checkpoint 有什么区别？

标准答案：checkpoint 记录的是 Agent 运行状态，比如当前步骤、计划、下一步动作和上下文摘要；工具执行记录记录的是某个具体工具动作是否已经发生、参数是什么、外部 request_id 是什么、结果是什么。

恢复时二者要配合。比如 checkpoint 显示下一步是创建工单，我不会直接再创建一次，而是先用 checkpoint 里的 idempotency_key 查工具执行记录。如果已经成功，就把原 ticket_id 写回 Agent state 继续；如果 unknown，就先对账；如果参数冲突，就停止并转人工。checkpoint 解决“从哪里继续”，工具执行记录解决“这个副作用是否已经发生”。

### 问题 10：面试官问“你怎么设计一个安全的退款 Agent”，你怎么答？

标准答案：我会先把退款工具定义为 critical 工具，Agent 不能直接自主执行。流程上，Agent 可以先调用只读工具查询订单、支付状态、可退款金额和用户权限，然后生成退款建议。系统侧做业务规则校验，比如订单归属、金额上限、是否重复退款、是否超过时效。

真正退款前必须让用户或审批人确认，确认后生成 approval_id，并和退款金额、订单号、用户、幂等键、参数 hash 绑定。执行器调用退款系统时写入工具执行记录，超时后进入 unknown 状态并先对账，不能直接重试。退款成功但通知或 CRM 同步失败时，用补偿队列处理后续步骤。最后所有步骤都要有 trace 和审计记录，方便排查和合规审查。

### 问题 11：如果模型重复选择同一个写工具，是 Prompt 问题还是工程问题？

标准答案：可能两者都有，但不能只靠 Prompt 修。Prompt 可以减少模型重复选择工具，比如告诉它已有结果后不要再调用；轨迹评估也可以发现重复调用问题。但真正防止事故的边界必须在工具执行层。

我的做法是：Agent state 里记录已完成工具，Prompt 中给模型必要摘要；同时执行器用幂等键和执行记录拦截重复副作用。这样即使模型又选择了一次 `create_ticket`，执行器也会返回已有 ticket_id，而不是真的再创建一个。面试里我会强调：模型行为要优化，系统边界也要兜住。

### 问题 12：如何在评测环境验证副作用工具不会真的执行？

标准答案：我会把工具分成只读、写操作和高风险操作。评测环境中，只读工具可以用录制数据或 Mock；写操作默认走 sandbox 或 dry-run；高风险工具必须 Mock，并且断言不会访问生产凭证和真实接口。

测试时不仅看最终答案，还要检查工具轨迹：是否调用了禁止工具，是否带了 idempotency_key，是否命中了人工确认要求，是否在超时后进入对账而不是重复执行。对于历史坏 Case，我会做回归样本，确保曾经出现过的重复工单、重复通知、重复退款不会再次发生。
