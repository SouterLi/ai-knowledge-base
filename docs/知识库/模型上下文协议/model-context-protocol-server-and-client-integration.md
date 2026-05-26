# AI 应用开发面试主题：MCP 服务端与客户端集成

## 主题选择记录

- 本次主题：MCP（Model Context Protocol）服务端与客户端集成
- 所属分类：工具生态与集成协议
- 已回顾历史主题：RAG、Agent 规划执行、Agent 工具调用、Prompt 工程、评估观测、安全防护、上下文工程、流式异步、多模态、成本缓存限流、模型网关、工作流编排、LLM 微调
- 不重复原因：已有工具调用主题聚焦模型输出函数调用与应用层执行闭环；本主题聚焦 **MCP 作为模型客户端与外部工具/资源之间的标准协议层**，重点考察协议抽象、服务端能力暴露、客户端发现、权限隔离和工程部署

---

## 一、为什么这是高频考点

AI 应用从「单个 Chat API」演进到「能访问文件、数据库、内部系统和第三方 SaaS」后，会遇到两个问题：

1. 每接一个工具都要为不同客户端重复写适配代码。
2. 工具、资源、权限、审计和上下文注入缺少统一协议。

MCP 的价值是把外部能力封装成标准服务，让 IDE、桌面助手、Agent 平台或企业 AI 网关都能用统一方式发现和调用这些能力。面试官问 MCP，通常不是考你背协议字段，而是看你是否理解 **“工具生态标准化 + 安全边界 + 可运维集成”**。

一句话回答：

> MCP 是一种让 AI 客户端以统一协议连接外部工具、数据资源和 Prompt 模板的上下文协议。它不是模型本身，也不是 Function Calling 的替代品，而是把可调用能力标准化暴露给 AI 应用的集成层。

---

## 二、核心概念

### 1. MCP 的三类角色

| 角色 | 职责 | 面试表述 |
| --- | --- | --- |
| MCP Host | 承载 AI 交互的应用，如 IDE、桌面助手、Agent 平台 | 负责管理用户会话、权限确认和模型交互 |
| MCP Client | Host 内部的协议客户端 | 与一个 MCP Server 建立连接，完成能力发现和调用 |
| MCP Server | 暴露工具、资源和 Prompt 的服务 | 把外部系统包装成标准协议能力 |

常见调用链：

```text
用户
  -> MCP Host（例如 IDE / Agent 平台）
  -> MCP Client
  -> MCP Server
  -> 本地文件 / 数据库 / SaaS API / 内部系统
```

面试中要强调：MCP Server 不直接“变聪明”，它只是把外部能力以模型友好的协议形式暴露出来；真正的推理和工具选择仍由 Host、模型和应用编排层完成。

### 2. Tools、Resources、Prompts

MCP Server 通常暴露三类能力：

| 能力 | 含义 | 适合场景 | 风险点 |
| --- | --- | --- | --- |
| Tools | 可执行动作，带输入参数和返回结果 | 查订单、发起搜索、执行内部 API | 权限、参数校验、副作用控制 |
| Resources | 可读取上下文资源，类似 URI 暴露的数据 | 文件、文档、日志、配置、数据库记录 | 数据泄露、越权访问、上下文过载 |
| Prompts | 可复用的提示词模板 | 标准分析流程、代码审查模板、客服回复模板 | 版本管理、提示注入、业务口径漂移 |

**面试必答：**

- Tool 是动作，Resource 是数据，Prompt 是可复用任务模板。
- Tool 调用必须做参数校验和权限校验。
- Resource 读取必须做访问控制和内容裁剪。
- Prompt 不能当安全边界，仍需服务端校验。

### 3. MCP 与 Function Calling 的关系

| 维度 | Function Calling / Tool Calling | MCP |
| --- | --- | --- |
| 关注点 | 模型如何输出结构化调用意图 | 客户端如何发现和连接外部能力 |
| 典型位置 | 模型 API 与应用编排层之间 | AI Host / Client 与外部服务之间 |
| 工具定义 | 通常随请求传给模型 | 由 MCP Server 对外暴露，Client 动态发现 |
| 执行方 | 应用层代码 dispatch | MCP Server 执行或代理到后端系统 |
| 核心价值 | 让模型输出可解析参数 | 让工具生态可复用、可发现、可治理 |

两者可以组合：

```text
模型选择调用某个工具
  -> Host 根据工具定义生成调用
  -> MCP Client 发送请求
  -> MCP Server 执行业务能力
  -> 结果回灌给模型
```

如果面试被问“MCP 会取代 Function Calling 吗”，回答应是：不会。Function Calling 更偏模型交互接口，MCP 更偏工具和上下文接入协议；实际系统经常同时使用。

---

## 三、核心知识点

### 1. MCP Server 的设计要点

一个可上线的 MCP Server 不只是注册几个函数，至少要包含：

1. **能力清单**：明确暴露哪些 tools、resources、prompts。
2. **Schema 契约**：每个 tool 的参数类型、必填字段、枚举值和返回结构。
3. **权限模型**：哪些用户、租户、项目可以调用哪些能力。
4. **执行边界**：是否允许写操作、是否有副作用、是否需要人工确认。
5. **审计日志**：记录调用来源、参数摘要、结果状态、耗时和错误码。
6. **错误协议**：参数错误、权限错误、上游失败、超时、限流要可区分。

面试高分表述：

> MCP Server 是 AI 能力接入层，不应把内部系统裸露给模型。它要把外部能力转换成“最小权限、强约束、可审计”的工具或资源。

### 2. Tool Schema 怎么写

Tool Schema 的核心目标是让模型更容易选对工具、填对参数，同时让服务端更容易拒绝危险输入。

反例：

```json
{
  "name": "query",
  "description": "查询数据",
  "inputSchema": {
    "type": "object",
    "properties": {
      "sql": { "type": "string" }
    }
  }
}
```

问题：直接暴露 SQL，模型可能生成危险查询，权限和审计也很难做。

更好的设计：

```json
{
  "name": "get_customer_ticket_summary",
  "description": "按工单 ID 查询当前用户有权限访问的客服工单摘要",
  "inputSchema": {
    "type": "object",
    "properties": {
      "ticket_id": {
        "type": "string",
        "description": "客服工单 ID，例如 TCK-20260522-001"
      },
      "include_internal_notes": {
        "type": "boolean",
        "description": "是否包含内部备注；仅管理员角色允许为 true"
      }
    },
    "required": ["ticket_id"]
  }
}
```

设计原则：

- 工具名使用业务动作，不使用 `run_sql`、`call_api` 这类万能工具。
- 参数尽量结构化，避免让模型拼接 SQL、Shell、URL。
- 枚举值优先于自由文本。
- 描述写清楚适用场景和限制。
- 写操作工具要明确副作用，并在 Host 侧触发用户确认。

### 3. Resources 的上下文控制

Resource 不是“把所有数据都塞给模型”。好的 Resource 设计要回答三个问题：

1. 用户是否有权读取？
2. 返回内容是否足够小、足够相关？
3. 内容中是否包含敏感信息或提示注入文本？

建议策略：

- 使用稳定 URI，例如 `ticket://TCK-001/summary`、`repo://project-a/file/src/main.py`。
- 默认返回摘要或片段，长文档分页读取。
- 对密钥、身份证、手机号等字段做脱敏。
- 为每次资源读取记录审计日志。
- 对来自外部文档的内容加边界标记，提醒模型“这是不可信数据”。

示例返回：

```json
{
  "uri": "ticket://TCK-20260522-001/summary",
  "mimeType": "application/json",
  "text": "{\"status\":\"open\",\"priority\":\"high\",\"summary\":\"用户反馈发票无法下载\"}"
}
```

### 4. Client 能力发现与调用流程

典型流程：

```text
1. Host 启动或用户授权后，创建 MCP Client
2. Client 与 MCP Server 建立连接
3. Client 拉取 tools/resources/prompts 能力清单
4. Host 根据用户任务和权限筛选可用能力
5. 模型决定是否使用某个工具或资源
6. Client 发起调用
7. Server 执行、记录审计并返回结构化结果
8. Host 将结果裁剪后回灌模型
```

面试中容易被追问：能力清单是否应该全部传给模型？

推荐回答：

> 不应该无脑全量传。应按任务场景、用户权限、租户配置和上下文预算筛选。工具太多会降低模型选择准确率，也会扩大攻击面。

### 5. 本地 stdio 与远程 HTTP/SSE

MCP Server 常见运行方式：

| 方式 | 特点 | 适合场景 |
| --- | --- | --- |
| stdio | Client 启动本地进程，通过标准输入输出通信 | 本地 IDE 插件、文件系统、开发工具 |
| HTTP/SSE | Server 独立部署，Client 通过网络连接 | 企业内部服务、团队共享工具、云端 Agent 平台 |

选型建议：

- 本地文件、Git、Shell 类能力优先 stdio，但要严格限制路径和命令范围。
- 企业 API、数据库、工单系统适合远程部署，便于鉴权、审计和统一升级。
- 远程模式必须考虑认证、TLS、租户隔离、限流和版本兼容。

### 6. 权限与安全边界

MCP 安全不是“写在工具描述里让模型遵守”，而是服务端和 Host 共同强制执行。

关键措施：

- **最小权限**：每个 Server 只暴露完成任务所需能力。
- **用户态授权**：以真实用户身份访问后端系统，不使用全局超级 Token。
- **危险操作确认**：删除、发送、支付、发布等写操作必须二次确认。
- **参数白名单**：路径、域名、枚举、租户 ID 都要校验。
- **结果脱敏**：工具返回前过滤敏感字段。
- **审计可追踪**：能回答“谁在什么上下文里让 AI 调了什么工具”。

典型反例：

```text
把企业数据库管理员账号放进 MCP Server，
再暴露一个 execute_sql(sql: string) 给所有用户。
```

正确做法是将能力拆成受限业务工具，例如 `get_my_orders`、`search_public_docs`、`create_support_ticket`，并在服务端按用户身份做权限过滤。

---

## 四、代码示例：一个最小 MCP 工具服务思路

下面是偏伪代码的 TypeScript 示例，用来表达工程结构，不绑定具体 SDK 版本。

```ts
type ToolInput = {
  ticket_id: string;
  include_internal_notes?: boolean;
};

type UserContext = {
  userId: string;
  role: "user" | "admin";
  tenantId: string;
};

async function getTicketSummary(input: ToolInput, user: UserContext) {
  if (!/^TCK-\d{8}-\d{3}$/.test(input.ticket_id)) {
    throw new Error("INVALID_TICKET_ID");
  }

  if (input.include_internal_notes && user.role !== "admin") {
    throw new Error("PERMISSION_DENIED");
  }

  const ticket = await ticketRepository.findById(input.ticket_id, user.tenantId);
  if (!ticket) {
    throw new Error("NOT_FOUND");
  }

  return {
    ticket_id: ticket.id,
    status: ticket.status,
    priority: ticket.priority,
    summary: ticket.summary,
    // 中文注释：内部备注只在管理员显式请求时返回，避免默认泄露敏感处理记录
    internal_notes: input.include_internal_notes ? ticket.internalNotes : undefined
  };
}

const getTicketSummaryTool = {
  name: "get_customer_ticket_summary",
  description: "按工单 ID 查询当前用户有权限访问的客服工单摘要",
  inputSchema: {
    type: "object",
    properties: {
      ticket_id: { type: "string" },
      include_internal_notes: { type: "boolean" }
    },
    required: ["ticket_id"]
  },
  handler: getTicketSummary
};
```

面试说明要点：

- Schema 约束的是模型输入格式，不能替代服务端校验。
- `tenantId`、`role` 不应由模型传入，应来自 Host 的认证上下文。
- 返回值要结构化，便于模型理解，也便于日志审计。
- 错误码要稳定，Host 才能决定重试、降级或提示用户授权。

---

## 五、工程落地 Checklist

### 1. 设计阶段

- [ ] 明确 Server 服务边界：只服务一个业务域，避免万能工具箱。
- [ ] 列出 tools/resources/prompts，标注只读、写入、敏感等级。
- [ ] 为每个工具定义输入 Schema、返回结构、错误码。
- [ ] 明确用户身份如何传递，租户隔离如何实现。
- [ ] 设计 Host 侧授权提示和危险操作确认。

### 2. 开发阶段

- [ ] 参数校验放在 Server 端，不依赖模型自觉。
- [ ] 所有外部 API 调用设置超时、重试和限流。
- [ ] 工具返回内容做裁剪、脱敏和结构化。
- [ ] 为只读工具和写操作工具分别设置审计字段。
- [ ] 对提示注入样本文档做测试，确认 Server 不执行文档中的恶意指令。

### 3. 上线阶段

- [ ] 灰度启用 MCP Server，先开放低风险只读工具。
- [ ] 监控调用量、错误率、P95 延迟、权限拒绝率。
- [ ] 记录工具版本，方便回滚和复盘。
- [ ] 为高风险工具设置 kill switch。
- [ ] 定期审查不再使用的工具，减少模型选择干扰和攻击面。

---

## 六、常见易错点

### 1. 把 MCP Server 做成万能代理

例如暴露 `http_request(url, method, body)` 或 `run_shell(command)`。这类工具看似灵活，实际会把 URL 访问、命令执行、数据泄露风险全部交给模型。

更好的做法：把高风险通用能力包装成有限业务动作，并在服务端做白名单。

### 2. 把工具描述当权限控制

“请不要访问敏感数据”只是提示，不是安全机制。模型可能误解，也可能被外部文档注入诱导。

权限必须由 Host 和 Server 基于真实用户身份强制执行。

### 3. 忽略工具数量对模型选择的影响

工具越多不一定越好。大量相似工具会导致模型误选、参数混淆和上下文浪费。

建议按任务动态加载工具，并保持工具命名和描述互斥清晰。

### 4. 返回结果过大

MCP Resource 或 Tool 返回几万行日志，会让模型丢失重点并增加成本。

建议返回摘要、分页、过滤后的字段，必要时提供二次读取工具。

### 5. 缺少审计

AI 调工具出了问题时，如果没有记录调用来源、参数、用户身份和工具版本，很难定位是模型误判、用户越权、工具 Bug 还是上游系统问题。

---

## 七、高频面试问题与参考答案

### Q1：MCP 是什么？解决什么问题？

**参考答案：**

MCP 是 Model Context Protocol，用于让 AI Host 以统一协议连接外部工具、资源和 Prompt 模板。它解决的是 AI 应用接入外部系统时重复适配、能力不可发现、权限和审计不统一的问题。它不是模型，也不是单纯的插件机制，而是模型应用和外部上下文之间的标准协议层。

### Q2：MCP 和 Function Calling 有什么区别？

**参考答案：**

Function Calling 关注模型如何输出结构化调用意图，通常发生在模型 API 和应用编排层之间；MCP 关注外部能力如何被 AI 客户端发现、连接和调用，通常发生在 Host/Client 与 MCP Server 之间。两者可以结合：模型通过 Function Calling 选择工具，Host 再通过 MCP Client 调用对应 Server。

### Q3：MCP Server 应该暴露哪些能力？

**参考答案：**

优先暴露边界清晰、权限可控、返回结构稳定的业务能力。MCP Server 可以暴露 tools、resources、prompts：tools 用于执行动作，resources 用于读取上下文，prompts 用于复用任务模板。不要暴露万能 SQL、万能 HTTP、万能 Shell 这类高风险工具，除非有非常严格的沙箱和白名单。

### Q4：如何设计一个安全的 MCP Tool？

**参考答案：**

首先工具要表达具体业务动作，而不是底层任意执行能力；其次输入 Schema 要尽量结构化，使用必填字段、枚举和格式约束；第三，服务端必须基于真实用户身份做权限校验，不能相信模型传来的用户 ID 或角色；第四，写操作要有用户确认、幂等设计和审计日志；最后，返回结果要脱敏、裁剪并使用稳定错误码。

### Q5：MCP Resource 和 RAG 文档检索有什么区别？

**参考答案：**

RAG 更关注如何从知识库中检索相关文档并增强生成，核心是索引、召回、重排和答案质量；MCP Resource 更像一种协议化的数据读取能力，可以暴露文件、日志、配置、数据库记录等上下文资源。两者可以结合：RAG 系统可以作为 MCP Server 的一个 resource 或 tool 暴露给 Host。

### Q6：远程 MCP Server 上线时要考虑什么？

**参考答案：**

要考虑认证、TLS、租户隔离、限流、超时、审计、版本兼容和灰度发布。远程 Server 往往连接企业内部系统，不能只按“插件”思路处理。建议先开放只读工具，监控错误率和权限拒绝率，再逐步开放写操作，并为高风险工具设置 kill switch。

### Q7：MCP 工具太多会有什么问题？怎么治理？

**参考答案：**

工具太多会增加上下文成本，让模型更容易误选工具，也会扩大攻击面。治理方式包括按任务动态加载工具、按用户权限过滤工具、合并重复工具、为工具建立清晰命名规范、监控工具使用率并下线长期不用的工具。

### Q8：如何防止 Prompt Injection 借 MCP 工具越权？

**参考答案：**

不能依赖提示词防御。服务端要把外部资源内容视为不可信输入，不能让文档里的指令改变工具权限；Host 侧要区分系统指令、用户指令和资源内容；Server 侧必须按真实用户身份做权限校验，对写操作做人类确认，对敏感返回做脱敏，并记录审计日志。

### Q9：MCP Server 出错时应该怎么返回？

**参考答案：**

错误要可分类、可恢复。参数错误返回稳定的校验错误，权限不足返回权限错误，上游超时返回超时错误，限流返回可重试信息。不要把内部堆栈直接返回给模型，也不要只返回“失败”。Host 需要根据错误类型决定让模型修正参数、提示用户授权、重试、降级或结束任务。

### Q10：如果让你给企业知识库做 MCP 接入，你会怎么设计？

**参考答案：**

我会做一个只读 MCP Server，暴露 `search_knowledge_base` 工具和 `kb://document/{id}` resource。搜索工具只接受关键词、业务域、时间范围等结构化参数，不允许模型直接写 SQL。Resource 读取按用户身份校验文档权限，并默认返回摘要和片段。所有调用记录用户、租户、查询条件、命中文档和耗时。上线时先灰度给内部团队，观察召回质量、权限拒绝率和敏感信息命中情况。

---

## 八、面试速记

- MCP 是 AI 应用连接外部工具、资源和 Prompt 的标准协议层。
- Host 管会话和授权，Client 管协议连接，Server 暴露能力。
- Tools 是动作，Resources 是数据，Prompts 是模板。
- MCP 不取代 Function Calling，二者常组合使用。
- 安全边界在 Host 和 Server，不在模型提示词。
- 不要暴露万能 SQL、万能 HTTP、万能 Shell。
- 能力清单要按任务、权限和上下文预算动态筛选。
- Tool Schema 只是第一层约束，服务端校验才是硬边界。
- 远程 Server 要重点考虑认证、租户隔离、审计、限流和版本治理。
- 高分答案要覆盖：协议抽象、权限控制、动态发现、错误处理、观测审计。
