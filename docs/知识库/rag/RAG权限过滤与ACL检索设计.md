# AI 应用开发面试主题：RAG 权限过滤与 ACL 检索设计

## 主题选择记录

- **本次主题序号**：第 35 篇。
- **README 位置**：`README.md` 目录表中追加到第 34 篇之后，归入 **RAG 工程化** 卷。
- **文件位置**：`docs/知识库/rag/RAG权限过滤与ACL检索设计.md`。
- **选题原因**：已有 RAG 总览、查询理解、答案生成与引用归因，也有身份权限与多租户隔离的安全总览，但「RAG 检索阶段如何做权限过滤、ACL 建模和跨租户防泄露」尚未单独展开。企业知识库、客服、内部搜索场景面试中，经常会追问“怎么避免用户搜到无权文档”“向量库不支持复杂权限怎么办”“权限变更后缓存和索引如何一致”，因此适合作为 RAG 工程化小主题补充。
- **去重结论**：本主题不重复已有篇章；它不泛讲鉴权体系，而是聚焦 RAG 的文档入库、向量召回、重排序、上下文组装、引用展示和缓存中的 ACL 约束。

## 核心概念

**RAG 权限过滤** 是指在检索增强生成链路中，基于服务端可信身份上下文，只召回、重排、注入和引用用户有权访问的文档或片段。它的目标不是让模型“不要说敏感信息”，而是在模型看到上下文之前，就把未授权数据挡在检索层外。

典型链路如下：

```text
用户请求
  -> 网关鉴权 / SSO / Token 解析
  -> 构造可信身份上下文
  -> 计算租户、角色、部门、文档组、敏感级策略
  -> 检索前 ACL 过滤或索引分区
  -> 授权候选召回
  -> rerank / 上下文压缩
  -> 只注入授权证据
  -> 答案生成、引用展示与审计
```

面试里要强调：**权限是系统硬边界，LLM 只是边界内的推理组件**。不能先召回全量文档再让模型保密，也不能依赖用户在自然语言里自报身份。权限判断必须来自认证链路、权限服务和数据层策略。

常见权限粒度：

| 粒度 | 示例 | 面试关注点 |
| --- | --- | --- |
| 租户级 | A 公司不能访问 B 公司知识库 | 是否强制带 `tenant_id` 或 namespace |
| 文档级 | 只有财务部可看报销制度 | ACL 是否随文档入库并可更新 |
| Chunk 级 | 同一文档部分段落涉密 | 解析与切分时是否继承或细化权限 |
| 字段级 | 结果中手机号、金额需脱敏 | 输出前是否再做敏感字段治理 |
| 操作级 | 可读知识库但不能下载原文 | 引用、预览、下载权限是否区分 |

## 核心知识点

### 1. 身份上下文必须由服务端生成

RAG 查询进入检索层前，应先把用户身份转换成稳定、可信、可审计的上下文。

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class IdentityContext:
    tenant_id: str
    user_id: str
    roles: tuple[str, ...]
    groups: tuple[str, ...]
    scopes: tuple[str, ...]
    permission_version: int


def build_identity_context(jwt_claims: dict) -> IdentityContext:
    # 身份只来自认证系统，不接受用户在问题中自称管理员
    return IdentityContext(
        tenant_id=jwt_claims["tenant_id"],
        user_id=jwt_claims["sub"],
        roles=tuple(jwt_claims.get("roles", [])),
        groups=tuple(jwt_claims.get("groups", [])),
        scopes=tuple(jwt_claims.get("scopes", [])),
        permission_version=int(jwt_claims.get("permission_version", 0)),
    )
```

面试中可以这样说：用户问题只是业务输入，不能参与决定“他是谁”。我会在网关或后端解析 Token，生成 `tenant_id`、`user_id`、`roles`、`groups`、`permission_version`，后续检索、缓存、审计都使用这份身份上下文。

### 2. 文档入库时就要写入 ACL 元数据

权限过滤不是在线查询时临时猜出来的。文档摄取、切分和索引构建阶段，就要把权限元数据写入文档、chunk 和索引。

```json
{
  "chunk_id": "hr_policy_2026_001_03",
  "doc_id": "hr_policy_2026_001",
  "tenant_id": "tenant_a",
  "title": "2026 年员工绩效制度",
  "text": "绩效申诉需要在结果公布后 5 个工作日内提交。",
  "acl": {
    "allow_groups": ["hr", "manager"],
    "allow_roles": ["hr_admin"],
    "deny_users": ["u_10086"],
    "sensitivity_level": 2
  },
  "doc_version": 7,
  "permission_version": 42
}
```

关键点有三个：

1. **权限继承**：chunk 默认继承文档权限，必要时细化到段落或字段。
2. **权限快照**：索引里记录文档版本和权限版本，方便追踪旧索引是否过期。
3. **默认拒绝**：权限缺失、解析失败、租户为空时不要放入可检索索引。

### 3. 检索前过滤优先于检索后过滤

更推荐在候选召回前就限定检索范围，而不是先召回 topK 再过滤。原因是检索后过滤会导致两个问题：第一，未授权内容已经被向量库或中间日志看到；第二，过滤后候选可能不足，召回质量不稳定。

```python
def search_authorized_chunks(query: str, identity: IdentityContext, top_k: int = 8) -> list[dict]:
    allowed_doc_ids = permission_service.list_allowed_documents(
        tenant_id=identity.tenant_id,
        user_id=identity.user_id,
        roles=identity.roles,
        groups=identity.groups,
    )

    if not allowed_doc_ids:
        return []

    return vector_store.search(
        query=query,
        top_k=top_k,
        namespace=identity.tenant_id,
        filter={
            "doc_id": {"$in": allowed_doc_ids},
            "sensitivity_level": {"$lte": policy.max_level(identity.roles)},
        },
    )
```

但也要说明现实限制：很多向量库的 metadata filter 对超大 `doc_id in (...)`、复杂布尔逻辑、层级部门权限支持不好。此时可以选择索引分区、预计算授权集合、倒排检索先裁剪、或者召回更多候选后做二次强校验。

### 4. 四种常见 ACL 检索方案

| 方案 | 做法 | 优点 | 风险 |
| --- | --- | --- | --- |
| 单索引 + metadata filter | 同一 collection，用 `tenant_id`、角色、文档组过滤 | 成本低，易运维 | 调用方漏传过滤条件会越权 |
| namespace / collection 隔离 | 按租户或数据域拆分索引 | 隔离更清晰 | 小租户多时索引碎片化 |
| 授权文档预计算 | 权限服务返回可访问 `doc_id` 集合 | 逻辑表达力强 | 权限变更要同步失效 |
| 候选召回后二次校验 | 召回较大 topK，再由权限服务裁剪 | 兼容弱过滤向量库 | 必须保证未授权候选不进 Prompt 和日志 |

面试回答不要绝对化。可以说：中小规模内部知识库可以从单索引加强制过滤起步；多租户 SaaS 通常按租户 namespace 隔离；强合规或大客户用独立索引；复杂组织权限则结合权限服务做预计算和二次校验。

### 5. Rerank、压缩和引用也要继承权限边界

权限过滤不是只发生在 vector search。只要数据可能进入模型、日志、引用或前端展示，就要继续保持授权边界。

```text
错误链路：
召回全量候选 -> rerank 看到无权 chunk -> 过滤后生成 -> 引用列表仍显示无权标题

正确链路：
授权过滤 -> 授权候选 rerank -> 授权上下文压缩 -> 授权引用展示 -> 审计
```

容易忽略的泄露点：

1. **reranker 输入**：跨编码器重排模型也不应看到无权文本。
2. **上下文压缩**：压缩器不能混入无权 chunk 中的摘要。
3. **引用标题**：即使不展示正文，文档标题也可能泄露信息。
4. **无权限提示**：不要说“你无权查看《裁员名单》”，应说“当前权限下没有可用依据”。
5. **日志与评测集**：脱敏后记录 `doc_id`、权限决策和哈希，不把敏感原文长期明文保存。

### 6. 缓存必须包含权限维度

RAG 缓存如果只按问题文本命中，很容易把 A 用户有权看到的答案返回给 B 用户。

```python
import hashlib
import json


def rag_cache_key(identity: IdentityContext, query: str, kb_version: int, prompt_version: str) -> str:
    # 缓存键必须包含权限版本，避免权限变更后继续命中旧答案
    payload = {
        "tenant_id": identity.tenant_id,
        "user_id": identity.user_id,
        "roles": sorted(identity.roles),
        "groups": sorted(identity.groups),
        "permission_version": identity.permission_version,
        "query": query.strip().lower(),
        "kb_version": kb_version,
        "prompt_version": prompt_version,
    }
    return hashlib.sha256(json.dumps(payload, ensure_ascii=False, sort_keys=True).encode()).hexdigest()
```

更稳妥的策略是：高风险场景不缓存最终答案，只缓存授权后的检索结果或 rerank 结果；权限变更、文档下线、知识库版本升级时，让相关缓存失效。

### 7. 权限变更与索引一致性

真实企业场景中，权限会频繁变化：员工转岗、离职、部门调整、文档密级升级。如果索引和缓存不更新，就可能继续泄露旧权限下的内容。

常见做法：

1. **权限版本号**：用户、文档或租户权限变化时递增 `permission_version`。
2. **事件驱动更新**：权限系统发送变更事件，触发索引 metadata 更新或缓存失效。
3. **查询时二次校验**：最终进入 Prompt 前，再按最新权限校验候选 chunk。
4. **延迟窗口控制**：如果索引异步更新，敏感权限收紧要优先失效缓存并临时走强校验。

面试里要主动提“权限收紧”和“权限放宽”的区别：权限放宽慢一点通常只是体验问题；权限收紧延迟则是安全事故。

### 8. 评测要覆盖越权与召回质量

只测答案准确率不够，ACL RAG 至少要有两类评测：

| 评测类型 | 样例 | 通过标准 |
| --- | --- | --- |
| 越权拦截 | 普通员工问高管薪酬制度 | 不召回、不引用、不暗示文档存在 |
| 同权限召回 | HR 问绩效申诉流程 | 能召回正确授权 chunk |
| 跨租户隔离 | A 租户问 B 租户产品文档 | 无任何 B 租户候选 |
| 权限变更 | 用户转出部门后复问旧问题 | 缓存失效，答案不再包含旧授权内容 |
| 引用安全 | 无权文档标题命中相似问题 | 前端引用列表不展示该标题 |

可以设计自动化断言：

```python
def assert_no_unauthorized_context(response: dict, forbidden_doc_ids: set[str]) -> None:
    returned_doc_ids = {item["doc_id"] for item in response.get("citations", [])}
    context_doc_ids = {item["doc_id"] for item in response.get("debug_context", [])}

    assert returned_doc_ids.isdisjoint(forbidden_doc_ids)
    assert context_doc_ids.isdisjoint(forbidden_doc_ids)
```

面试表达要补一句：ACL 评测既看安全，也看可用性。不能为了安全把召回范围收得过窄，导致有权限用户也查不到正确答案。

## 高频面试问题与标准答案

**Q1：RAG 系统怎么避免用户看到无权限文档？**

标准答案：我会把权限控制放在检索链路前面，而不是靠 Prompt。请求进来后先由后端解析身份，得到 tenant、user、roles、groups 和权限版本；文档入库时写入 ACL 元数据；检索时强制带 tenant 和 ACL filter，只召回授权 chunk；rerank、上下文压缩、引用展示都只处理授权候选。最后再记录审计日志，确保能追踪“谁在什么时候访问了哪些文档”。

**Q2：为什么不能先召回全量文档，再让模型不要泄露？**

标准答案：因为模型不是安全边界。只要无权文档进入召回候选、reranker、Prompt 或日志，就已经有泄露风险，而且提示词可能被注入绕过。正确做法是在数据进入模型前就过滤掉，模型只能在授权上下文内回答。面试里我会强调：权限必须由服务端、检索层和数据层保证，Prompt 只能做辅助说明。

**Q3：向量库的 metadata filter 不支持复杂 ACL，怎么办？**

标准答案：我会先看权限复杂度和数据规模。简单租户隔离可以用 namespace 或 collection；复杂组织权限可以由权限服务预计算可访问 doc_id，再缩小检索范围；如果 `doc_id in` 太大，就用倒排索引先做授权裁剪，或者分数据域建索引。对于过滤能力弱的向量库，可以召回更大候选后做服务端二次强校验，但必须保证未授权候选不进入 Prompt、引用和可持久日志。

**Q4：多租户知识库用一个索引还是多个索引？**

标准答案：没有固定答案。中小租户可以单索引加 tenant filter，成本低；租户数量多但隔离要求中等，可以按 namespace 或 collection；强合规、大客户、私有化部署更适合独立索引。无论哪种方案，都要避免调用方漏传 tenant_id，最好在检索 SDK 或服务端统一注入，而不是让业务代码手写过滤条件。

**Q5：权限变更后，RAG 如何避免继续命中旧答案？**

标准答案：我会引入权限版本号。用户角色、文档 ACL、部门关系变化时递增 `permission_version`，缓存 key 包含这个版本；文档权限收紧时触发缓存失效和索引 metadata 更新；最终进入 Prompt 前再按最新权限二次校验。尤其是权限收紧，宁可短时间降低命中率，也不能继续暴露旧权限内容。

**Q6：RAG 缓存为什么会带来权限风险？**

标准答案：如果缓存 key 只有 query，A 用户问“年终奖规则”得到的授权答案，B 用户可能用同一句问题命中同一个答案。缓存 key 至少要包含 tenant、user 或权限范围、permission_version、知识库版本、Prompt 版本。敏感场景我更倾向于缓存授权后的检索结果，或者不缓存最终自然语言答案。

**Q7：用户没有权限时，答案应该怎么说？**

标准答案：不要泄露资源存在性。比如不能说“你无权访问《高管薪酬调整方案》”，因为标题本身就是敏感信息。更合适的表达是“当前权限下没有可用于回答该问题的知识库依据”，必要时提示联系管理员申请权限。这样既说明无法回答，又不暴露无权文档。

**Q8：ACL 过滤会不会影响召回效果？怎么平衡安全和效果？**

标准答案：会影响，所以要同时评测安全和可用性。安全侧看越权文档是否零进入 Prompt、引用和日志；效果侧看有权限用户的 Recall@K、MRR、答案准确率是否下降。工程上可以通过合理的权限粒度、索引分区、授权集合预计算、混合检索和 rerank 来减少误伤，但原则是安全边界不能为了召回率被放松。

**Q9：如何设计 ACL RAG 的测试用例？**

标准答案：我会构造同一问题、不同身份的对照集。例如 HR 能查绩效制度，普通员工只能查公开版制度；A 租户永远不能召回 B 租户文档；用户转岗后复问旧问题必须不命中旧缓存。断言不只看最终回答，还要检查 debug context、citations、日志摘要里没有 forbidden doc_id。

**Q10：面试中一句话总结 RAG 权限过滤的关键是什么？**

标准答案：RAG 权限过滤的关键是“可信身份前置、授权候选召回、权限版本贯穿缓存和索引、无权数据绝不进入模型上下文”。我不会把它当成 Prompt 问题，而是当成检索系统和数据安全问题来设计。
