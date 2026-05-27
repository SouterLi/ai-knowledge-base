# LLM 流式响应与异步任务架构

## 核心概念

### 1. 两种响应模式

| 模式 | 特点 | 适用 |
| --- | --- | --- |
| 同步一次性返回 | 实现简单，等待完整生成 | 短问答、低延迟要求不高 |
| 流式响应 | 边生成边返回，改善 TTFT | 在线聊天、写作助手 |

长任务（报告生成、批量分析、长文档总结）应使用**异步任务**：API 创建任务，Worker 执行，前端轮询/SSE/WebSocket 看进度。

### 2. 关键指标

- **TTFT**：首 token 时间，决定“是否有反应”。
- **端到端延迟**：完整答案结束时间。
- **吞吐与并发**：受模型限流、连接数、队列影响。

### 3. 通信协议选型

| 协议 | 特点 | 场景 |
| --- | --- | --- |
| SSE | 单向推送，浏览器友好 | 聊天流式 |
| WebSocket | 双向 | 语音、协作、中途控制 |
| 轮询 | 简单稳定 | 长任务状态 |

生产环境**不让前端直连模型 API**——后端负责鉴权、限流、Prompt、过滤、日志与成本。

---

## 核心知识点

### 1. 结构化流式协议

用事件类型区分内容与状态，勿只推纯文本：

| 事件 | 含义 |
| --- | --- |
| `delta` | 新增文本片段 |
| `metadata` | 模型、引用、用量、task_id |
| `error` | 可展示错误 |
| `done` | 生成结束 |

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()

async def stream_answer():
  for chunk in ["正在分析...", "核心结论是...", "完成"]:
    # 中文注释：SSE 用 event + data，空行分隔
    yield f"event: delta\ndata: {chunk}\n\n"
  yield "event: done\ndata: {}\n\n"

@app.get("/chat/stream")
def chat_stream():
  return StreamingResponse(stream_answer(), media_type="text/event-stream")
```

### 2. 后端中转职责

鉴权 → 限流 → Prompt 组装 → 调用上游 → 敏感过滤 → 结构化事件 → 审计/计费。  
网关/Nginx **禁止缓冲** `text/event-stream`。

### 3. 用户取消

前端断连 ≠ 上游停止。后端需：

- 监听客户端断开，**取消上游请求**；
- 释放连接，标记 `cancelled`；
- 异步 Worker 在安全检查点读取消标记并退出。

```python
async def proxy_stream(upstream, disconnect_event):
  async for chunk in upstream:
    if disconnect_event.is_set():
      await upstream.aclose()  # 中文注释：断连后停止消耗 token
      break
    yield chunk
```

### 4. 部分输出后失败

发送 `error` 事件，前端提示“结果不完整”；关键业务**先草稿后提交**，不边流边落库最终结果。

### 5. 异步任务架构

```text
API 校验 → 写任务表(pending) → 投递队列 → Worker 执行
       → 更新 running/progress/succeeded/failed
       → 前端轮询/SSE/WebSocket 获取进度
```

任务字段：`task_id`、`user_id`、`status`、`progress`、`result_url`、`error_code`、时间戳。

```python
def create_task(payload: dict, idempotency_key: str) -> str:
  existing = task_repo.find_by_key(idempotency_key)
  if existing:
    return existing.task_id  # 中文注释：重试返回同一任务，避免重复执行
  task_id = task_repo.insert(status="pending", payload=payload)
  queue.publish(task_id)
  return task_id
```

### 6. 幂等与重试

创建任务与写操作带 `idempotency_key`；Worker 重试需识别已完成步骤；失败任务支持**安全重放**或人工介入。

### 7. 观测指标

TTFT、总耗时、失败率、取消率、平均 token、队列等待时间、chunk 间隔 P95。

### 8. 何时异步化

超过 HTTP 超时、依赖多外部系统、结果需下载、批量处理——返回 `task_id` + 进度，而非长时间挂连接。

---

## 高频面试问题与标准答案

**Q1：流式为什么需要后端中转？**

统一鉴权、限流、Prompt、敏感过滤、日志与成本；隐藏 API Key；转换供应商差异的 chunk 格式。

**Q2：用户中途取消怎么办？**

监听断连 → 取消上游 → 标记 cancelled；异步任务 Worker 轮询取消标记；记录取消率分析体验问题。

**Q3：流式中途失败如何处理？**

协议发 `error`；UI 标明不完整；关键写操作未完成前不提交；可保留已生成草稿供用户决定。

**Q4：如何避免重复执行？**

`idempotency_key` 创建任务；写工具幂等键；队列消费至少一次时用状态机防重。

**Q5：SSE 和 WebSocket 怎么选？**

聊天单向输出优先 SSE；需客户端中途发控制（打断、切换模式、语音）用 WebSocket。

**Q6：流式能降低模型推理时间吗？**

不能减少完整 decode 时间，主要降低 TTFT 和感知延迟；完整耗时仍受输出 token 影响。

**Q7：结构化 JSON 如何流式？**

边流边解析或等 `done` 再校验；半截 JSON 不可直接进下游；或先流自然语言再单独结构化步骤。

**Q8：长任务架构怎么答？**

入口 API、队列、Worker、状态存储、进度通知、失败重试、幂等、观测——按此展开。

---

## 面试回答加分点

1. 区分**同步/流式/异步**三种模式及选型依据（超时、依赖数、交互需求）。
2. 强调**结构化事件协议**，而非 raw text pipe。
3. 取消链路：**客户端 → 网关 → 应用 → 上游模型** 全链路传递。
4. 高风险业务：**展示 ≠ 执行完成**（工具/写库须等真实结果）。
5. 异步任务提**幂等键 + 状态机**，体现可靠性工程。
6. 观测同时看 TTFT 与**队列等待**，定位是模型慢还是排队慢。
7. 架构题按“入口、队列、Worker、存储、通知、幂等、观测”七段回答，条理清晰。
