# MCP 服务端与客户端集成

## 核心概念

### 1. MCP 是什么

**Model Context Protocol**：让 AI Host 以统一协议连接外部**工具（Tools）**、**资源（Resources）**、**提示模板（Prompts）**。解决多客户端重复适配、能力不可发现、权限审计不统一——不是模型本身，也不是 Function Calling 的替代品。

### 2. 三类角色

| 角色 | 职责 |
| --- | --- |
| MCP Host | IDE/Agent 平台；会话、授权、模型交互 |
| MCP Client | Host 内协议客户端；连接 Server、发现与调用 |
| MCP Server | 暴露 tools/resources/prompts；代理到文件/DB/SaaS |

```text
用户 → Host → Client → Server → 后端系统
```

### 3. Tools / Resources / Prompts

| 能力 | 含义 | 风险 |
| --- | --- | --- |
| Tools | 可执行动作（查单、搜库） | 权限、副作用、参数校验 |
| Resources | 可读上下文（文件、记录 URI） | 泄露、上下文过大 |
| Prompts | 可复用任务模板 | 注入、口径漂移 |

- Tool = 动作；Resource = 数据；Prompt ≠ 安全边界。

### 4. MCP 与 Function Calling

| | Function Calling | MCP |
| --- | --- | --- |
| 层次 | 模型输出调用意图 | 客户端连接外部能力 |
| 工具定义 | 常随请求传入 | Server 暴露，Client 发现 |
| 组合 | 模型选工具 → Host → MCP Client → Server 执行 → 结果回灌 |

---

## 核心知识点

### 1. MCP Server 设计要点

能力清单、Schema 契约、权限模型、写操作确认、审计日志、稳定错误码（参数/权限/超时/限流可区分）。

### 2. Tool Schema：反模式与正例

反例：`query` + 自由 `sql` 字符串。  
正例：业务动作 + 结构化参数。

```json
{
  "name": "get_customer_ticket_summary",
  "description": "按工单 ID 查询当前用户有权限的工单摘要",
  "inputSchema": {
    "type": "object",
    "properties": {
      "ticket_id": { "type": "string", "description": "如 TCK-20260522-001" },
      "include_internal_notes": { "type": "boolean" }
    },
    "required": ["ticket_id"]
  }
}
```

原则：工具名表意、枚举优于自由文本、写操作标明副作用、描述写清限制。

### 3. Resources

稳定 URI（`ticket://ID/summary`）；默认摘要/分页；脱敏；不可信内容边界标记防注入。

### 4. Client 发现流程

连接 → 拉取能力清单 → **按任务/权限/上下文预算筛选** → 模型决策 → 调用 → 裁剪后回灌。  
不应把全部 tools 无脑传给模型。

### 5. 传输方式

| 方式 | 场景 |
| --- | --- |
| stdio | 本地 IDE、文件、Git；须严格路径/命令白名单 |
| HTTP/SSE | 企业 API、共享工具；需 TLS、认证、租户隔离、限流 |

### 6. 安全边界

最小权限、用户态 Token（非全局超管）、危险操作二次确认、参数白名单、返回脱敏、全链路审计。

### 7. 代码示例（TypeScript 思路）

```ts
type ToolInput = { ticket_id: string; include_internal_notes?: boolean };
type UserContext = { userId: string; role: "user" | "admin"; tenantId: string };

async function getTicketSummary(input: ToolInput, user: UserContext) {
  if (!/^TCK-\d{8}-\d{3}$/.test(input.ticket_id)) {
    throw new Error("INVALID_TICKET_ID");
  }
  if (input.include_internal_notes && user.role !== "admin") {
    throw new Error("PERMISSION_DENIED"); // 中文注释：角色由 Host 注入，不信模型传的 role
  }
  const ticket = await repo.findById(input.ticket_id, user.tenantId);
  if (!ticket) throw new Error("NOT_FOUND");
  return {
    ticket_id: ticket.id,
    status: ticket.status,
    summary: ticket.summary,
    internal_notes: input.include_internal_notes ? ticket.internalNotes : undefined,
  };
}
```

---

## 高频面试问题与标准答案

**Q1：MCP 解决什么问题？**  
统一 AI 应用接入外部工具/资源/模板的方式；减少重复适配；统一发现、权限与审计。

**Q2：与 Function Calling 区别？**  
FC 是模型侧结构化意图；MCP 是 Host/Client 与 Server 间的集成协议；常组合使用。

**Q3：Server 应暴露什么？**  
边界清晰的业务 tools、只读 resources、版本化 prompts；避免万能 SQL/HTTP/Shell。

**Q4：如何设计安全 Tool？**  
具体业务动作、结构化 Schema、服务端基于真实用户校验、写操作确认与幂等、脱敏返回、稳定错误码。

**Q5：Resource 与 RAG？**  
RAG 重检索增强生成；Resource 是协议化读取能力；RAG 可作为 Server 的 tool/resource 暴露给 Host。

**Q6：远程 Server 上线？**  
认证、TLS、租户隔离、限流、超时、审计、版本兼容；先只读灰度，写操作 kill switch。

**Q7：工具太多？**  
误选、成本高、攻击面大；动态加载、权限过滤、合并重复、下线闲置工具。

**Q8：防 Prompt Injection？**  
资源内容不可信；系统/用户/资源分层；服务端硬权限；写操作人审。

**Q9：错误如何返回？**  
可分类、可恢复；不泄露堆栈；Host 决定重试/授权/降级。

**Q10：企业知识库 MCP 设计？**  
只读 Server：`search_knowledge_base`（结构化参数）+ `kb://doc/{id}` resource；权限校验；默认摘要；全链路审计；先内部灰度。

---

## 面试回答加分点

1. **定位**：MCP 是「能力接入层」，不是把 DB 管理员账号交给模型。  
2. **反对万能代理**：`http_request(url)`、`run_shell(cmd)` 反例要讲清。  
3. **Schema ≠ 安全**：描述里的「请勿越权」无效，Host+Server 强制执行。  
4. **动态工具集**：按任务加载，体现模型选择准确率意识。  
5. **返回裁剪**：大日志分页/摘要，控 token 与幻觉。  
6. **审计可答**：谁、何时、何工具、何参数摘要、何版本。  
7. **与 Agent 编排**：MCP 提供能力，规划/重试仍在 Host 应用层。  
8. **速记七条**：Host-Client-Server、三类能力、与 FC 组合、安全在服务端、stdio vs 远程、治理工具数量、观测审计。
