# LLM 流式响应与异步任务架构

## 主题选择记录

- **主题**：LLM 流式响应与异步任务架构
- **分类**：AI 应用开发 / Runtime
- **仓库去重**：不重复 RAG、Agent、Prompt 内容；聚焦**运行时链路、首字延迟、长任务**
- **适用岗位**：AI 应用开发、后端/全栈工程师

## 核心概念

LLM 应用常见两种交付方式：

1. **同步一次性返回**：实现简单，用户干等，易触网关超时。  
2. **流式响应**：边生成边推送（SSE/WebSocket/chunked），改善 **TTFT（首 token 时间）** 和交互感。

还要区分 **TTFT**、**端到端延迟**、**并发吞吐**。报告生成、批量分析、长文档总结等往往超过 HTTP 超时，应改为**异步任务**：接口只创建任务，Worker 执行，前端轮询或订阅进度。

生产里**不让前端直连模型**：鉴权、限流、Prompt 组装、脱敏、日志、成本统计、错误转换都在后端。

## 核心知识点

### 1. 协议选型

| 协议 | 特点 | 场景 |
|------|------|------|
| SSE | 单向推送，浏览器友好 | 聊天流式 |
| WebSocket | 双向 | 语音、协作、中途打断/改参 |
| 轮询/订阅任务 | 稳定、易重试 | 长任务、离线任务 |

### 2. 流式事件协议（不要只推纯文本）

建议分事件类型，前端好处理状态：

```python
# SSE 示例：delta / metadata / error / done
async def stream_answer(task_id: str):
    yield f"event: metadata\ndata: {{\"task_id\": \"{task_id}\"}}\n\n"
    for chunk in llm_stream():
        yield f"event: delta\ndata: {json.dumps({'text': chunk})}\n\n"
    yield "event: done\ndata: {}\n\n"
```

错误用 `event: error`，别和正文拼在一个字符串里让用户猜。

### 3. FastAPI 流式入口示例

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()

@app.get("/chat/stream")
async def chat_stream():
    return StreamingResponse(
        stream_answer("t-123"),
        media_type="text/event-stream",
        headers={"Cache-Control": "no-cache", "X-Accel-Buffering": "no"},
    )
```

`X-Accel-Buffering: no` 避免 Nginx 缓冲导致「假流式」。

### 4. 异步任务架构

1. API 校验权限 → 写任务表 `pending`  
2. 投递队列（Redis/Celery/Kafka 等）  
3. Worker 执行，更新 `running/progress/succeeded/failed`  
4. 客户端轮询、`GET /tasks/{id}` 或 SSE 推进度  

关键字段：`task_id, user_id, status, progress, result_url, error_code, idempotency_key, created_at, updated_at`。

```python
# 创建任务 + 幂等
def create_task(user_id: str, payload: dict, idem_key: str | None):
    if idem_key and (existing := find_by_idem(idem_key)):
        return existing
    task = insert_task(user_id, payload, idem_key)
    queue.publish(task.id)
    return task
```

### 5. 取消与断连

用户关页面 ≠ 上游停止计费。后端监听 disconnect，取消模型 HTTP 请求；异步任务写 `cancelled`，Worker 在安全检查点退出。

### 6. 部分失败

流式已展示一半再失败：发 `error` 事件，UI 标「不完整」；关键业务**先草稿后提交**，不要边流式边写生产库最终态。

### 7. 幂等与重试

创建任务、写操作带 `idempotency_key`；客户端重试返回同一 `task_id`，避免双倍扣费或双倍执行。

### 8. 观测

TTFT、总耗时、取消率、流式中断率、队列等待时间、Worker 失败重试、token 用量。按路由分维度（聊天 vs 报告）。

### 9. 何时必须异步

超过网关超时（常 30～60s）、多外部依赖串联、结果需生成文件下载、需后台重试——都应异步，聊天短问答才优先流式同步。

## 高频面试问题与标准答案

### 1. 流式和同步怎么选？

用户等待的短问答用流式，体验好。超长生成、批量任务、要断点续跑的一律异步。我会用产品超时线和 P99 耗时来划界，而不是凭感觉。

### 2. TTFT 为什么重要？

用户感知「系统有没有反应」主要看首字多久出来。TTFT 高会像卡死，哪怕总时间和非流式一样。优化方向包括模型选型、减少串行预处理、边检索边生成（要注意质量）。

### 3. 为什么需要后端中转？

要鉴权、租户限流、拼 RAG、过滤敏感输出、记 token 成本、统一错误码。前端直连密钥也泄露，没法做审计。

### 4. SSE 和 WebSocket 怎么选？

纯聊天推 token 用 SSE 足够，实现简单。需要客户端中途发「停止」「换模型」或双向音频，再用 WebSocket。别为了酷全上 WS 增加运维成本。

### 5. 用户刷新/关页面怎么停模型？

监听连接断开，abort 上游 HTTP stream；异步任务则更新 cancelled 并让 Worker 协作式退出。否则后台还在烧 token。

### 6. 流式一半失败怎么处理？

协议层发 error；UI 保留已展示内容并提示不完整。财务、合同类结果等 `done` 再落库，避免半截脏数据。

### 7. 异步任务如何保证不重复执行？

`idempotency_key` + 任务状态机；Worker 处理前 CAS 把 `pending` 改成 `running`。写操作业务层也要幂等。

### 8. 队列积压怎么扩容？

看等待时间告警，水平扩 Worker；区分优先级队列；非核心任务降级。同时查上游模型 429，别无限加 Worker 放大限流。

### 9. 进度条怎么设计靠谱？

Worker 上报阶段性 `progress`（0～100 或步骤名），别只靠前端假动画。长任务还要存中间 artifact URL，失败可重试从某步恢复。

### 10. 面试里如何讲清整条架构？

我会按：入口 API（鉴权/限流）→ 同步流式 or 创建任务 → 队列 → Worker（调模型/工具）→ 状态存储 → 进度通知（SSE/轮询）→ 幂等/取消/观测。这样结构清晰，也显得做过生产。
