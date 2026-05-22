# AI 应用开发面试主题：LLM 应用身份权限与多租户隔离

## 主题选择记录

- 本次主题：LLM 应用中的身份识别、权限控制与多租户数据隔离。
- 所属分类：AI 应用开发 / Security
- 已回顾历史主题：RAG 系统设计、Embedding 与向量索引、GraphRAG、Agent 工具调用、Agent 规划执行、Prompt 工程、结构化输出、MCP 集成、多模态、LLM 微调、上下文记忆、Text-to-SQL / NL2SQL、流式异步、测试 Mock、LLM 工作流编排与 Human-in-the-loop、模型网关、成本缓存限流、发布配置治理、评估观测、坏 Case 归因、安全防护。
- 不重复原因：已有安全主题聚焦提示注入、越权访问和敏感数据泄露的通用防线；Text-to-SQL 主题重点在 SQL 生成、校验和查询执行；Workflow 主题重点在流程编排、人工介入和状态治理。本文专门展开“身份上下文如何贯穿 LLM 调用链，以及租户、角色、数据权限如何落到检索层、工具层和审计层”。
- 适用岗位：AI 应用工程师、LLM 平台工程师、RAG/Agent 工程师、后端架构师、AI 安全工程师。

## 一、为什么这是高频考点

很多候选人做 Demo 时只关心“模型能不能回答”，但企业级 LLM 应用更常见的问题是：同一个模型服务给多个租户、多个角色、多个业务线使用，如何保证它只看见当前用户有权访问的数据，只调用当前用户有权执行的工具，并且出了问题能追溯。

面试官问这个主题，通常不是想听“加一个登录态”这么浅的答案，而是在考察你是否理解：

1. LLM 不是权限系统，不能让模型自己判断用户能不能访问数据。
2. 权限要贯穿请求入口、检索召回、Prompt 组装、工具调用、输出返回和日志审计。
3. 多租户隔离不是只在数据库表加 `tenant_id`，还包括向量索引、缓存、文件对象、模型上下文、异步任务和观测日志。

一句话：身份权限是 LLM 应用的系统边界，模型只是边界内的推理组件。

## 二、核心概念

### 1. 身份上下文

身份上下文是一次 LLM 请求中用于权限判断的最小可信信息集合，通常包括：

- `user_id`：当前用户。
- `tenant_id`：当前租户或组织。
- `roles`：角色，如普通成员、管理员、客服主管。
- `scopes`：细粒度权限，如 `ticket:read`、`refund:write`。
- `data_groups`：可访问的数据范围，如部门、项目、知识库集合。
- `request_id`：链路追踪和审计用的请求标识。

关键点：身份上下文必须由服务端从登录态、Token 或网关认证结果中解析，不能相信用户自然语言输入里的身份声明。

### 2. 多租户隔离

多租户隔离是指多个客户、组织或业务线共享同一套 LLM 应用基础设施时，数据、配置、调用额度、日志和权限互不串扰。

常见隔离层次：

- 逻辑隔离：同一数据库或向量库，通过 `tenant_id`、权限过滤和策略控制隔离。
- 索引隔离：不同租户使用不同 collection、namespace 或 index。
- 物理隔离：大客户、强合规场景使用独立数据库、独立对象存储、独立模型部署。

面试回答要结合风险选择方案：逻辑隔离成本低但对工程正确性要求高；物理隔离成本高但合规边界清晰。

### 3. 权限前置

权限前置是指在数据进入模型上下文之前就完成访问控制，而不是让模型拿到全量数据后再“自觉不说”。

典型例子：

- RAG：先按 `tenant_id` 和文档 ACL 过滤，再做向量召回或混合检索。
- Agent：先判断用户是否有工具调用权限，再把工具 schema 暴露给模型。
- 工作流：先过滤用户可触发的节点，再让模型补全参数。

## 三、核心知识点

### 1. 推荐调用链

```text
用户请求
  -> API 网关鉴权
  -> 应用服务构造身份上下文
  -> 权限策略引擎判断可访问资源
  -> 检索层按权限过滤候选文档
  -> Prompt 组装层只注入授权上下文
  -> 工具执行层再次校验权限
  -> 输出安全检查和脱敏
  -> 审计日志记录身份、资源、工具和结果摘要
```

这里最重要的是“双重校验”：检索前校验数据权限，工具执行前校验操作权限。不要把权限判断只放在 Prompt 模板里。

### 2. RAG 场景的数据权限

RAG 权限设计至少要覆盖三个阶段：

1. 入库阶段：文档切分后的每个 chunk 必须携带权限元数据。
2. 召回阶段：查询向量库时带上租户和 ACL 过滤条件。
3. 组装阶段：Prompt builder 只拼接通过权限校验的片段。

示例元数据：

```json
{
  "chunk_id": "doc_9527_chunk_03",
  "tenant_id": "tenant_a",
  "doc_id": "policy_2026",
  "allowed_scopes": ["knowledge:hr:read", "knowledge:finance:read"],
  "allowed_roles": ["hr_admin", "manager"],
  "allowed_user_ids": ["u_1001"],
  "department_ids": ["hr", "finance"],
  "sensitivity_level": 2
}
```

如果向量库不支持足够强的过滤能力，可以采用“先查授权文档 ID，再限定向量召回范围”的方式，避免跨租户召回。

### 3. Agent 工具权限

Agent 工具调用要区分“能不能看见工具”和“能不能执行工具”：

- 工具暴露前：根据用户权限生成可用工具列表，普通用户不应看到管理员工具。
- 参数生成后：后端再次校验参数是否越权，比如不能查询其他租户的订单。
- 执行前：高风险操作需要确认、审批、幂等键和审计。
- 执行后：工具返回值要做字段级裁剪，避免把敏感字段塞回模型上下文。

一个成熟系统不会给模型一个 `run_sql(sql)` 或 `http_request(url)` 万能工具，而是提供窄接口工具，例如 `get_my_ticket_status(ticket_id)`、`create_refund_request(order_id, reason)`。

### 4. 缓存与多租户

LLM 应用常见缓存包括 Prompt 缓存、RAG 检索缓存、模型响应缓存、工具结果缓存。多租户场景下，缓存 key 必须包含权限相关维度：

```text
cache_key = hash(
  tenant_id,
  user_id 或 role_scope_version,
  model_alias,
  prompt_template_version,
  retrieval_policy_version,
  normalized_query
)
```

如果缓存 key 只包含用户问题，就可能出现 A 租户问过的问题被 B 租户命中，导致数据泄露。

### 5. 审计与可追溯

审计日志不是把完整 Prompt 全量落盘。更合理的做法是记录可追溯但不过度暴露隐私的信息：

- 谁发起：`tenant_id`、`user_id`、角色、请求来源。
- 访问了什么：文档 ID、工具名、资源 ID、权限策略版本。
- 为什么允许或拒绝：命中的策略、拒绝原因。
- 模型做了什么：模型别名、Prompt 版本、工具参数摘要、输出摘要。
- 结果如何：成功、拒绝、失败、人工确认、降级。

审计日志要支持按请求链路追踪，也要支持按用户、租户、工具和资源反查。

## 四、策略与代码示例

### 1. 权限策略示例

```yaml
tenant_policy:
  isolation_mode: "namespace"
  require_tenant_filter: true
  deny_cross_tenant_cache: true

rag_policy:
  required_metadata:
    - tenant_id
    - allowed_scopes
    - allowed_roles
    - allowed_user_ids
    - sensitivity_level
  retrieval_filter: "tenant_id == current.tenant_id AND allowed_scopes intersects current.scopes"

tool_policy:
  refund_order:
    allowed_scopes: ["refund:write"]
    require_human_confirm: true
    max_amount: 1000
  search_employee_salary:
    allowed_roles: ["hr_admin"]
    mask_fields: ["id_card", "bank_account"]
```

### 2. 服务端权限校验伪代码

```python
def build_rag_context(query: str, identity: IdentityContext) -> list[Chunk]:
    # 中文注释：身份上下文来自服务端认证结果，不接受用户在问题中自报身份
    allowed_doc_ids = permission_service.list_allowed_documents(
        tenant_id=identity.tenant_id,
        user_id=identity.user_id,
        scopes=identity.scopes,
    )

    # 中文注释：召回时带上租户和授权文档范围，避免敏感内容进入模型上下文
    chunks = vector_store.search(
        query=query,
        namespace=identity.tenant_id,
        filter={
            "doc_id": {"$in": allowed_doc_ids},
            "sensitivity_level": {"$in": identity.allowed_sensitivity_levels},
        },
        top_k=8,
    )
    return chunks


def execute_tool(tool_name: str, arguments: dict, identity: IdentityContext) -> ToolResult:
    # 中文注释：模型只能提出调用意图，真正的权限判断必须由后端执行
    policy = tool_policy_registry.get(tool_name)
    if not policy.allows(identity, arguments):
        raise PermissionDenied(f"tool denied: {tool_name}")

    sanitized_arguments = policy.sanitize(arguments)
    result = tool_executor.run(tool_name, sanitized_arguments)
    return policy.mask_result(result, identity)
```

这段伪代码面试时可以直接解释成三句话：身份来自服务端；检索时做租户和 ACL 过滤；工具执行前后都做权限与脱敏。

## 五、高频面试问题与参考答案

### 1. LLM 应用里为什么不能只依赖 Prompt 做权限控制？

Prompt 只能影响模型行为，不能作为安全边界。模型可能被提示注入诱导，也可能误解复杂权限规则。真正的权限控制必须由服务端、检索层、工具执行层和数据层完成。正确做法是：服务端构造身份上下文；数据进入模型前先做权限过滤；工具执行前再次校验；输出前做敏感信息检查和脱敏。

### 2. RAG 系统如何避免跨租户数据泄露？

要从入库、召回、组装和缓存四层防护。入库时给每个 chunk 写入 `tenant_id` 和 ACL 元数据；召回时使用租户 namespace 或 metadata filter；Prompt 组装只使用通过校验的 chunk；缓存 key 包含租户、角色或权限版本。不能先召回全量数据再要求模型不要泄露，因为敏感内容一旦进上下文，风险就已经发生。

### 3. 多租户向量库应该用一个 index 还是多个 index？

取决于规模、隔离要求和运维成本。中小租户、权限模型一致时，可以用同一个 index 加 `tenant_id` 过滤或 namespace 隔离，成本低、扩缩容简单。大客户、强合规或数据量差异很大时，可以使用独立 index，隔离更强，也方便单独调参和迁移。面试中要补一句：无论哪种方案，都要测试过滤条件是否强制生效，不能让调用方漏传 `tenant_id`。

### 4. Agent 工具调用如何做权限控制？

分四步：第一，根据身份上下文只暴露用户可用的工具；第二，模型生成参数后，后端校验参数范围和资源归属；第三，高风险写操作加人工确认、审批或二次校验；第四，工具返回结果做字段级脱敏。核心原则是模型可以决策“想调用什么”，但不能决定“是否被允许调用”。

### 5. 如果用户在问题里说“我是管理员”，系统应该怎么处理？

把这句话当作普通不可信输入，不改变真实身份上下文。管理员身份只能来自服务端认证结果，例如登录态、企业 SSO、JWT claims 或网关鉴权信息。模型可以理解用户的问题，但权限系统只能相信可信认证链路。

### 6. LLM 响应缓存为什么会带来权限风险？

如果缓存 key 只按问题文本生成，不包含租户、用户、角色、权限版本和 Prompt 版本，就可能把一个用户的授权答案返回给另一个无权用户。解决方案是把权限维度纳入缓存 key；敏感问题不缓存或只缓存脱敏结果；权限变更时让相关缓存失效。

### 7. 如何设计权限变更后的数据一致性？

权限变更后要处理三类旧数据：检索索引中的 ACL 元数据、缓存中的历史结果、异步任务中的旧身份上下文。常见做法是引入权限版本号 `permission_version`，每次权限变更递增；召回和缓存 key 包含版本号；长任务执行前重新校验权限；敏感场景下主动刷新索引或删除旧缓存。

### 8. 面试官让你设计一个企业知识库问答系统的权限模型，怎么答？

可以按链路回答：

1. 文档入库时绑定租户、部门、文档 owner、可见角色、敏感等级。
2. 用户请求进入后，服务端解析身份上下文和可访问范围。
3. 检索阶段按租户 namespace 和 ACL 过滤后再向量召回。
4. Prompt 只注入授权 chunk，并保留引用来源。
5. 输出阶段做敏感字段检测和引用校验。
6. 记录审计日志，包括用户、文档、策略版本和请求结果。

这样回答比单纯说“数据库加权限字段”更完整。

## 六、容易踩坑的回答

- 只说“系统提示词里告诉模型不要泄露数据”：安全边界放错了。
- 只在最终输出过滤敏感信息：敏感数据已经进入上下文，仍可能被间接利用。
- 缓存 key 不带租户和权限：高概率造成跨用户或跨租户串数据。
- 给 Agent 暴露万能工具：权限、审计和参数约束都会失控。
- 只记录模型输出日志：事后无法解释模型为什么访问某个文档或调用某个工具。
- 把 `tenant_id` 当成前端传参：容易被篡改，应该由服务端认证链路注入。

## 七、实践检查清单

- [ ] 请求入口是否由服务端构造身份上下文？
- [ ] RAG chunk 是否包含租户、ACL、敏感等级等元数据？
- [ ] 检索过滤是否默认强制带 `tenant_id`？
- [ ] Prompt 组装是否只使用授权数据？
- [ ] Agent 工具是否按权限动态暴露并在执行前二次校验？
- [ ] 工具返回结果是否做字段级脱敏？
- [ ] 缓存 key 是否包含租户、权限版本、Prompt 版本和策略版本？
- [ ] 权限变更后是否能让缓存、索引和异步任务感知？
- [ ] 审计日志是否能追溯用户、资源、策略和工具调用？

## 八、一分钟总结

LLM 应用的身份权限设计，本质是把传统后端的认证、授权、多租户隔离和审计能力，延伸到检索、Prompt、工具调用和缓存这些 LLM 特有环节。面试时抓住三句话：身份来自服务端认证，不来自用户自然语言；数据进入模型前必须先做权限过滤；模型能提出调用意图，但真正的工具执行和数据访问必须由后端权限系统裁决。
