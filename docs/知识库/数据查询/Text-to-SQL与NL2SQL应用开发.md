# Text-to-SQL / NL2SQL 应用开发

## 核心概念

### 1. Text-to-SQL 是什么

把自然语言问题转为可执行、可审计的数据库查询。工程重点不是「写出 SQL」，而是**语义层、Schema Linking、校验、权限、执行治理与评测**。

```text
用户问题 → 意图/澄清 → 表字段召回 → 指标匹配 → 生成 SQL → AST 校验 → 权限注入 → Explain/Dry Run → 执行 → 解释/可视化
```

### 2. Schema Linking

把「新用户」「渠道」「留存率」「上周」映射到物理表、字段或指标公式。语法正确但业务错，多半是 Linking 失败。

### 3. 语义层

把物理字段封装为指标、维度、过滤口径，避免模型临时「发明公式」。

```yaml
metrics:
  retention_rate_7d:
    name: 7日留存率
    expression: retained_users_7d / new_users
    dimensions: [channel, signup_date]
    filters: [is_test_user = false]
```

### 4. SQL 方言与安全

Prompt/配置必须声明 dialect（MySQL/PG/BigQuery 等）。安全不能靠 Prompt：禁止 DDL/DML、限制表白名单、只读账号、行级权限、扫描量与超时。

---

## 核心知识点

### 1. 推荐架构（反模式：全库 DDL + 直接执行）

```text
对话入口 → 意图识别 → 数据目录/语义层检索 → Prompt 构造 → LLM 生成 → SQL Parser → 权限改写 → Explain → 执行 → 审计
```

### 2. Schema 召回

关键词 + 向量 + 血缘 + 历史相似 SQL；召回后**权限过滤**，只把少量相关表字段放进 Prompt。

### 3. Prompt 要素

方言、当前时间、相关表字段与样例值、可用指标、只读/租户/LIMIT 约束、缺口径时返回 `need_clarification`。

### 4. AST 校验（优于纯正则）

```python
def validate_sql(ast, allowed_tables, allowed_columns):
    # 中文注释：只允许 SELECT；表/字段必须在白名单内
    if ast.statement_type != "SELECT":
        raise ValueError("只允许 SELECT")
    for table in ast.collect_tables():
        if table not in allowed_tables:
            raise PermissionError(f"无权访问表: {table}")
    for table, col in ast.collect_columns():
        if col not in allowed_columns.get(table, set()):
            raise PermissionError(f"无权访问字段: {table}.{col}")
```

### 5. 权限注入

租户/部门条件由服务端基于 AST 注入，不依赖模型在 SQL 里「自觉写 tenant_id」；数据库层 RLS/只读账号双保险。

### 6. 成本控制

Explain 估扫描量、默认 LIMIT、禁止无分区大表全扫、超时、结果截断、重复问题缓存。

### 7. 多轮追问

维护**结构化查询状态**（指标、时间、过滤、维度），由构建器重生成 SQL，而非拼接历史 SQL 字符串。

```json
{
  "metric": "gmv",
  "time_range": "last_7_days",
  "dimensions": ["channel"],
  "filters": {"region": "华东"},
  "comparison": "previous_period"
}
```

### 8. 错误修复循环

语法/方言错误可给模型脱敏错误信息重试；权限/超时不能让模型绕过；勿把完整 DB 错误暴露给用户。

### 9. 评测

Execution Accuracy、Schema Linking 命中率、Business Acceptance、Clarification Rate；不只看 Exact Match（等价 SQL 很多）。

---

## 高频面试问题与标准答案

**Q1：最大难点？**  
业务语义到物理 Schema 的映射；需数据目录、语义层、权限与执行校验，不是 SQL 语法。

**Q2：为何不能把全库 Schema 塞进 Prompt？**  
上下文长、干扰多、易泄露无权限对象、难维护；应召回少量相关对象。

**Q3：如何防危险 SQL？**  
AST 只允许单条 SELECT；表白名单；服务端注入租户；只读账号；Explain/超时；审计日志。

**Q4：与 RAG 关系？**  
RAG 检索表说明、指标定义、历史 SQL 作上下文；最终仍须校验与执行。

**Q5：语义层为何重要？**  
统一「活跃用户」「转化率」等口径，可审计、可评测。

**Q6：执行报错怎么办？**  
分类：语法/字段/权限/超时/方言；权限类拒绝；超时则改写或缩小范围；错误信息脱敏后再给模型修复。

**Q7：如何评估？**  
离线：可执行率、执行结果一致、字段命中、澄清准确率；线上：成功率、慢查询、采纳率、纠错率。

**Q8：多轮怎么做？**  
结构化状态机更新后重生成 SQL，便于解释与回放。

**Q9：「上周」怎么处理？**  
服务端按业务时区、周起始日、数据延迟解析为结构化时间范围，再传给生成器。

**Q10：防敏感数据泄露？**  
字段分级；召回与 AST 阶段过滤；库层行列权限；敏感字段拒绝或脱敏审批。

**Q11：何时澄清？**  
缺指标定义、时间范围、分子分母时主动澄清，优于生成错误 SQL。

**Q12：灰度上线？**  
先低风险只读、少量数据集；展示 SQL+解释；沉淀高频样例；评测稳定再扩域。

---

## 面试回答加分点

1. **定位成数据平台能力**：模型生成候选，系统负责语义、安全、执行、评测。  
2. **开放题八步**：入口 → 语义层 → 生成 → 校验 → 执行 → 反馈/闭环。  
3. **反模式点名**：全库 DDL 直跑、正则当安全、模型传 tenant_id。  
4. **Text-to-SQL + Agent**：工具应是受控查询接口，不是裸 `execute_sql`。  
5. **成本意识**：Explain、分区、LIMIT 是上线必选项。  
6. **澄清优于瞎猜**：体现产品成熟度。  
7. **混合架构**：复杂分析可结合 RAG 文档说明指标定义。
