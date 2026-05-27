# LLM 应用身份权限与多租户隔离

## 核心概念

### 1. 身份上下文

一次 LLM 请求中用于权限判断的**服务端可信信息**：

- `user_id`、`tenant_id`、`roles`、`scopes`、`data_groups`、`request_id`

**必须由服务端从 Token/SSO/网关解析**，不能相信用户自然语言里的“我是管理员”。

### 2. 多租户隔离

多客户共享基础设施时，数据、配置、额度、日志、缓存**互不串扰**。层次：

| 方式 | 特点 | 适用 |
| --- | --- | --- |
| 逻辑隔离 | 同库 + tenant_id 过滤 | 成本低，工程要求高 |
| 索引隔离 | 不同 namespace/collection | 中等隔离 |
| 物理隔离 | 独立库/部署 | 强合规、大客户 |

### 3. 权限前置

在数据**进入模型上下文之前**完成访问控制：RAG 先过滤再召回；Agent 先判权再暴露工具 schema；工作流先过滤可触发节点。

**金句**：身份权限是系统边界，模型只是边界内的推理组件。

---

## 核心知识点

### 1. 推荐调用链

```text
网关鉴权 → 构造身份上下文 → 策略引擎 → 检索层 ACL 过滤
        → Prompt 仅注入授权数据 → 工具二次校验 → 输出脱敏 → 审计
```

**双重校验**：检索前校验数据权限；工具执行前校验操作权限。

### 2. RAG 数据权限

```json
{
  "chunk_id": "doc_03",
  "tenant_id": "tenant_a",
  "allowed_scopes": ["knowledge:hr:read"],
  "allowed_roles": ["hr_admin"],
  "sensitivity_level": 2
}
```

向量库过滤能力不足时：**先查授权 doc_id 列表，再限定向量召回范围**。

```python
def build_rag_context(query: str, identity) -> list:
  # 中文注释：身份来自认证，不接受用户在问题中自报身份
  allowed_ids = permission_service.list_documents(
    tenant_id=identity.tenant_id,
    user_id=identity.user_id,
    scopes=identity.scopes,
  )
  return vector_store.search(
    query=query,
    namespace=identity.tenant_id,
    filter={"doc_id": {"$in": allowed_ids}},
    top_k=8,
  )
```

### 3. Agent 工具权限

| 阶段 | 动作 |
| --- | --- |
| 暴露前 | 按权限生成可用工具列表 |
| 参数生成后 | 校验资源归属与参数范围 |
| 执行前 | 高风险：确认/审批/幂等 |
| 执行后 | 字段级脱敏再回注上下文 |

```python
def execute_tool(name: str, args: dict, identity) -> dict:
  policy = tool_policy_registry.get(name)
  if not policy.allows(identity, args):
    raise PermissionDenied(name)
  result = tool_executor.run(name, policy.sanitize(args))
  return policy.mask_result(result, identity)
```

### 4. 缓存与多租户

```python
def cache_key(identity, prompt_ver: str, query: str) -> str:
  # 中文注释：必须含 tenant、权限版本，否则 A 租户答案泄漏给 B
  return hash((
    identity.tenant_id,
    identity.permission_version,
    prompt_ver,
    normalize(query),
  ))
```

### 5. 权限变更一致性

引入 `permission_version`，变更时递增；召回与缓存 key 含版本；长任务执行前**重新校验**；敏感场景主动失效缓存。

### 6. 审计

记录：谁（tenant/user/role）、访问什么（doc_id/tool/资源）、为何允许/拒绝、模型做了什么（别名、Prompt 版本、参数摘要）。**不全量存 Prompt**。

---

## 高频面试问题与标准答案

**Q1：为什么不能只靠 Prompt 做权限？**

Prompt 可被注入绕过，不是安全边界。权限须在服务端、检索层、工具层、数据层实现；数据进模型前过滤，工具执行前后校验。

**Q2：RAG 如何避免跨租户泄露？**

入库写 tenant+ACL；召回强制 tenant 过滤；Prompt 只用授权 chunk；缓存 key 含租户与权限版本。不能召回全量再要求模型保密。

**Q3：多租户用一个 index 还是多个？**

中小租户：单 index + tenant 过滤/namespace，成本低。大客户/强合规：独立 index。无论哪种，**测试过滤是否强制生效**，禁止调用方漏传 tenant_id。

**Q4：Agent 工具权限怎么做？**

动态暴露工具 → 参数校验 → 写操作确认 → 返回脱敏。模型决定“想调用什么”，后端决定“是否允许”。

**Q5：用户说“我是管理员”怎么处理？**

当作不可信输入；管理员身份只来自 JWT/SSO/网关 claims。

**Q6：响应缓存为何有权限风险？**

key 若只有问题文本，可能把 A 用户授权答案给 B。解决：key 含 tenant、permission_version、prompt_version；敏感问题不缓存。

**Q7：设计企业知识库权限模型？**

文档入库绑部门/角色/敏感级 → 解析身份 → 检索 ACL 过滤 → Prompt 带引用 → 输出敏感检测 → 审计可追溯。

---

## 面试回答加分点

1. 三句话收尾：**身份来自认证；数据进模型前过滤；工具执行由后端裁决**。
2. 主动提 **tenant_id 不能由前端传参**，应由认证链路注入。
3. 区分“看见工具”和“执行工具”两阶段权限。
4. 缓存、异步任务、索引 ACL 与**权限版本**一并考虑，体现全链路思维。
5. 对比逻辑隔离 vs 物理隔离的**成本与合规**取舍。
6. 审计支持**按 request_id 正向追踪**和**按用户/资源反向查询**。
7. 避免踩坑：只在最终输出过滤、给 Agent 万能工具、缓存无租户维度。
