## 主题选择记录

- **本次序号**：第 48 篇。
- **README 位置**：追加到目录表末尾，归入「生产化与 LLMOps」卷，文件放在 `docs/知识库/提示词工程/` 下。
- **选题理由**：README 已有「Prompt 工程与结构化输出」和「发布、配置与实验治理」，但面试里经常继续追问：**提示词改了怎么上线？怎么做版本管理、灰度、回滚和效果评估？Prompt 变更导致线上效果变差怎么排查？多团队共用提示词时如何治理？** 本篇把 PromptOps 作为生产化专题单独展开。
- **避免重复**：不重复讲提示词写法、结构化输出解析、普通配置发布和 A/B 实验平台；本文只聚焦 **提示词资产管理、版本语义、评审流程、离线评测、灰度发布、回滚、观测和权限治理**。

## 核心概念

### 1. 什么是 PromptOps

PromptOps 是把提示词从「写在代码里的字符串」升级为可版本化、可评审、可测试、可灰度、可回滚、可观测的生产资产管理体系。它关注的不只是 prompt 怎么写得更好，而是 prompt 作为线上行为的一部分，如何安全持续迭代。

一句面试表达：**Prompt 工程解决“怎么写出好提示词”，PromptOps 解决“好提示词如何在生产环境里稳定发布、持续验证和快速回滚”。**

典型链路如下：

```text
需求变更
  → Prompt 草稿
  → 版本登记与差异审查
  → 离线评测集回归
  → 小流量灰度 / A/B 实验
  → 线上指标与坏 Case 观察
  → 全量发布或回滚
```

### 2. 为什么提示词需要版本管理

提示词会直接影响模型输出的口径、格式、工具调用意图、拒答策略和成本。如果没有版本管理，线上排障会很困难：

- 不知道某个坏 Case 是哪版 prompt 生成的。
- 改了系统提示词后，下游 JSON Schema 解析突然失败。
- RAG 引用要求变了，但评测集没有覆盖，幻觉率上升。
- 多个业务线复制粘贴 prompt，修复一个漏洞后其他场景仍然有风险。
- 模型升级、工具 schema 变更和 prompt 变更混在一起，无法归因效果波动。

面试中要强调：**prompt_version 应该像 model_version、index_version 一样进入 trace 和评测记录，否则无法复现、比较和回滚。**

### 3. PromptOps 和普通配置管理的区别

| 维度 | 普通配置 | PromptOps |
| --- | --- | --- |
| 变更影响 | 多数是确定性逻辑 | 影响概率性输出和模型行为 |
| 验证方式 | 单元测试、集成测试 | 离线评测集、人工抽检、线上坏 Case |
| 风险类型 | 配错阈值、开关误开 | 幻觉、格式漂移、越权、拒答异常、成本上升 |
| 回滚依据 | 错误率、服务指标 | 质量指标、解析成功率、引用正确率、用户反馈 |
| 依赖关系 | 服务版本、环境变量 | 模型版本、工具 schema、知识库版本、输出协议 |

所以 PromptOps 不能只用一个配置中心字符串替代。它需要把「提示词内容」「变量模板」「适用场景」「模型约束」「评测结果」「发布状态」一起管理。

### 4. 提示词版本的核心矛盾

PromptOps 的核心矛盾是 **迭代速度、输出质量、线上稳定性、治理成本**：

- 改得太快：业务响应敏捷，但容易引入隐性行为变化。
- 流程太重：质量更稳，但团队会绕过平台直接改代码。
- 只看离线分数：可能忽略真实用户长尾问题。
- 只看线上反馈：发现问题太晚，且难以复现。
- 只管理系统 prompt：忽略 RAG 模板、工具说明、输出 schema 和 few-shot 示例。

成熟方案通常是分层治理：低风险文案类 prompt 可以快速发布；影响工具调用、权限、医疗金融建议、结构化输出协议的 prompt 必须经过评测、审批和灰度。

## 核心知识点

### 1. 提示词资产应该如何建模

一个可落地的 prompt registry 不只保存文本，还要保存元数据和发布状态：

```sql
CREATE TABLE prompt_registry (
    prompt_id TEXT PRIMARY KEY,
    scenario TEXT NOT NULL,
    prompt_name TEXT NOT NULL,
    prompt_version TEXT NOT NULL,
    template TEXT NOT NULL,
    variables_json JSON NOT NULL,
    output_schema TEXT,
    model_family TEXT NOT NULL,
    owner_team TEXT NOT NULL,
    risk_level TEXT NOT NULL,
    status TEXT NOT NULL,
    created_by TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL
);
```

关键字段：

| 字段 | 面试关注点 |
| --- | --- |
| `scenario` | 区分客服、知识库问答、SQL 生成、Agent 工具选择等场景 |
| `prompt_version` | 用于评测对比、灰度和回滚 |
| `variables_json` | 约束模板变量，防止运行时缺字段或注入未经处理的内容 |
| `output_schema` | 和结构化输出解析绑定，避免 prompt 改了但 parser 没改 |
| `model_family` | 同一 prompt 在不同模型上效果可能不同 |
| `risk_level` | 决定是否需要审批、人工抽检和更严格评测 |
| `owner_team` | 线上坏 Case 要能找到负责人 |

### 2. 版本号应该表达什么

提示词版本不一定要完全照搬语义化版本，但至少要能表达变更影响：

```text
prompt_id: rag_answer_zh
version: 2.3.0

2: 输出契约或行为策略有破坏性变化
3: 示例、约束、拒答策略等能力增强
0: 文案修复、拼写修正、低风险微调
```

面试中可以这样说：如果只是修正文案或增加一个 few-shot，我会走小版本；如果改变 JSON 字段、引用格式、工具调用条件或安全拒答边界，就按破坏性变更处理，并要求下游 parser、评测集和回滚方案同步更新。

### 3. Prompt 模板渲染要可控

生产系统不建议到处拼字符串，而应使用受控模板渲染，并对变量做校验、截断和转义。

```python
from string import Template


def render_prompt(template: str, variables: dict, required_keys: set[str]) -> str:
    # 中文注释：缺少变量时直接失败，避免模型收到半截模板后产生不可控输出
    missing = required_keys - variables.keys()
    if missing:
        raise ValueError(f"missing prompt variables: {sorted(missing)}")

    safe_variables = {}
    for key, value in variables.items():
        text = str(value)
        # 中文注释：用户输入统一截断，真实系统还应做脱敏和注入风险标记
        safe_variables[key] = text[:4000]

    return Template(template).substitute(safe_variables)
```

面试重点不是具体用哪个模板引擎，而是说明三点：**变量白名单、长度控制、渲染失败可观测**。如果模板缺变量还继续调用模型，后续坏 Case 很难归因。

### 4. 发布前必须跑离线评测

Prompt 改动上线前，至少要用固定评测集做回归。评测集通常包括：

| 类型 | 示例 | 指标 |
| --- | --- | --- |
| 黄金样本 | 高频真实问题与标准答案 | 准确率、关键信息覆盖率 |
| 坏 Case 回归 | 历史幻觉、越权、格式失败样本 | 修复率、回归失败数 |
| 边界样本 | 模糊问题、缺字段、超长输入 | 拒答正确率、澄清率 |
| 安全样本 | Prompt injection、越权请求 | 拦截率、泄露率 |
| 格式样本 | JSON、SQL、工具参数 | 解析成功率、Schema 通过率 |

离线评测不能只看平均分。更适合面试的说法是：我会设置发布门禁，例如解析成功率不能下降、P0 安全样本必须全过、核心业务集不能低于上一版本，同时对成本和延迟做对比。

### 5. 灰度发布和流量切分

Prompt 灰度通常按租户、用户、场景或请求 hash 切流，而不是随机修改所有流量。

```python
def select_prompt_version(user_id: str, scenario: str, rollout: dict) -> str:
    # 中文注释：相同用户稳定命中同一版本，避免一次会话里行为来回变化
    key = f"{scenario}:{user_id}"
    bucket = hash(key) % 100

    if bucket < rollout["canary_percent"]:
        return rollout["candidate_version"]
    return rollout["stable_version"]
```

灰度观察指标包括：

- 输出质量：点赞率、人工评分、任务完成率、引用正确率。
- 稳定性：解析失败率、重试率、拒答率、工具调用失败率。
- 安全性：越权拦截、敏感信息泄露、注入攻击命中。
- 成本体验：输入 token、输出 token、TTFT、总耗时。
- 业务结果：转人工率、解决率、SQL 执行成功率、Agent 完成率。

面试中要补一句：**灰度最好保持会话级粘性**。否则同一个用户上文由 v1 解释，下文由 v2 接续，可能导致口径不一致。

### 6. 回滚不只是切回旧文本

Prompt 回滚要同时考虑依赖项：

```text
prompt_version
  ↔ output_schema / parser_version
  ↔ model_version
  ↔ tool_schema_version
  ↔ index_version
  ↔ eval_dataset_version
```

如果 v2 prompt 新增了 `confidence` 字段，而下游 parser 已经按新字段改造，单独切回 v1 可能继续失败。因此发布记录要保存一组可恢复配置：

```json
{
  "release_id": "rag-answer-2026-06-24",
  "prompt_version": "2.3.0",
  "model_version": "gpt-x-2026-06",
  "parser_version": "answer-json-v4",
  "index_version": "kb-2026-06-20",
  "tool_schema_version": "search-v2"
}
```

面试表达：我会把 prompt 回滚当成一次配置组合回滚，而不是只把数据库里的 prompt 文本改回去。

### 7. 线上观测必须带 prompt_version

每次模型调用都应该记录：

| 字段 | 用途 |
| --- | --- |
| `trace_id` | 串联一次用户请求 |
| `prompt_id` / `prompt_version` | 对比不同版本效果 |
| `model_version` | 区分模型变更影响 |
| `template_variables_hash` | 避免直接记录敏感变量，同时支持归因 |
| `output_schema` | 排查结构化解析失败 |
| `eval_tags` | 标记场景、风险等级、是否命中坏 Case |
| `quality_feedback` | 用户反馈、人工审核、自动评分 |

没有这些字段，线上出现“今天回答变差了”只能靠猜。比较专业的排障路径是：先按 prompt_version 聚合质量指标，再看模型版本、检索版本、输入分布和工具失败率是否同步变化。

### 8. 多团队协作和权限治理

当多个团队共用 PromptOps 平台时，需要明确权限边界：

- 业务团队可以提交候选 prompt，但不能绕过高风险门禁直接全量发布。
- 平台团队维护模板渲染、安全扫描、评测流水线和发布能力。
- 安全或合规团队负责高风险场景的策略审核。
- 线上值班人员可以执行回滚，但回滚动作必须审计。
- Prompt 里涉及系统策略、工具说明、权限约束的部分应限制编辑权限。

面试中可以说：我会把 prompt 拆成可治理的层次，例如系统策略层、任务说明层、场景模板层、few-shot 示例层。业务可以改示例和口径，但安全边界和工具权限不能由业务随意改。

### 9. 常见事故与处理方式

| 事故 | 可能原因 | 处理方式 |
| --- | --- | --- |
| JSON 解析失败上升 | prompt 修改了字段名或输出说明 | 回滚 prompt/parser 组合，补格式评测 |
| 幻觉率上升 | 删除了引用约束或 few-shot 误导 | 恢复引用要求，加入坏 Case 回归 |
| 工具误调用增加 | 工具描述过宽或示例不清 | 收紧工具触发条件，增加负例 |
| 成本突然上涨 | prompt 变长、few-shot 过多 | 统计输入 token，压缩系统提示和示例 |
| 拒答率异常 | 安全策略写得过严 | 按场景分析拒答样本，拆分风险规则 |
| 不同租户口径混乱 | prompt 未按场景或租户隔离 | 引入 scenario、tenant 和版本隔离 |

面试里不要只说“回滚”。更好的回答是：先止血回滚，再用 trace 找到具体版本和样本，补到评测集，最后调整发布门禁，避免同类问题再次发生。

### 10. PromptOps 和 CI/CD 的结合

成熟团队会把 prompt 变更接入类似代码发布的流程：

```text
提交 prompt 变更
  → 静态检查：变量、长度、敏感词、禁止语
  → 离线评测：质量、安全、格式、成本
  → 人工评审：高风险场景审批
  → 生成 release 记录
  → 灰度发布
  → 指标达标后全量
```

可以把 prompt 存在 Git、数据库或专门平台里，但关键不是存储位置，而是是否具备 **差异审查、自动评测、发布记录、灰度策略和审计回滚**。

## 高频面试问题与标准答案

**Q1：PromptOps 和 Prompt 工程有什么区别？**

标准答案：Prompt 工程更偏“怎么写”，比如角色、约束、示例、结构化输出；PromptOps 更偏“怎么管和怎么上线”，包括版本管理、评测、灰度、回滚、监控和权限。线上系统里 prompt 会直接影响质量、安全和成本，所以我会把它当成生产资产治理，而不是代码里的一个字符串。

**Q2：为什么 prompt 改动也要做版本管理？**

标准答案：因为 prompt 改动会改变模型行为，而且这种变化不是完全确定性的。如果没有 prompt_version，线上坏 Case 很难复现，也无法判断质量下降是 prompt、模型、检索还是工具变化导致的。我会把 prompt_version 写入 trace、评测记录和发布记录，和 model_version、index_version 一起用于归因和回滚。

**Q3：一个 prompt 上线前你会怎么验证？**

标准答案：我会先跑离线评测集，覆盖高频样本、历史坏 Case、安全样本、边界问题和格式样本。指标上不只看平均分，还会看解析成功率、引用正确率、拒答正确率、幻觉率、成本和延迟。通过门禁后再小流量灰度，并按 prompt_version 观察线上质量和错误率，确认没问题再扩大流量。

**Q4：Prompt 灰度发布怎么做？**

标准答案：一般按租户、用户、场景或请求 hash 切流，并保持会话级粘性，避免同一轮对话里 prompt 版本来回切换。灰度期间我会对比候选版本和稳定版本的任务成功率、解析失败率、用户反馈、工具调用失败率、token 成本和安全拦截指标。如果核心指标变差，就停止放量并回滚到稳定版本。

**Q5：Prompt 回滚是不是把文本切回旧版本就可以？**

标准答案：不一定。prompt 往往和模型版本、输出 schema、parser、工具 schema、知识库版本一起工作。如果新 prompt 改了 JSON 字段，下游 parser 也跟着改了，单独切回旧 prompt 可能仍然出错。所以我会保存 release 维度的配置组合，回滚时切回一组验证过的 prompt、parser、模型和相关依赖。

**Q6：线上发现某个版本回答质量下降，你怎么排查？**

标准答案：我会先按 prompt_version 聚合指标，确认是不是某个版本开始变差；然后看同一时间是否有模型、检索索引、工具 schema 或流量分布变化。接着抽取坏 Case，看是幻觉、格式漂移、拒答异常还是工具误调用。处理上先降级或回滚止血，再把样本加入评测集，修复 prompt 或门禁后重新灰度。

**Q7：Prompt 变更如何避免破坏结构化输出？**

标准答案：我会把 output_schema 和 prompt_version 绑定，发布前跑格式评测，确保 JSON 字段、类型、枚举值和必填项都能通过 parser。对关键场景，prompt 中会明确“只输出 JSON，不要解释”，但不能只依赖提示词，还要在服务端做 schema 校验、自动修复或失败兜底。只要 schema 有破坏性变化，就按大版本处理。

**Q8：多团队共用 PromptOps 平台时怎么治理？**

标准答案：我会区分编辑、评审、发布和回滚权限。业务团队可以提交候选 prompt 和 few-shot，但高风险场景要经过评测和审批；平台团队维护模板、安全扫描、评测和发布能力；安全边界、工具权限、系统策略不能随意改。所有发布和回滚都要有 owner、变更原因、评测结果和审计记录。

**Q9：Prompt 存 Git 好还是存数据库好？**

标准答案：两种都可以，取决于团队流程。Git 的优势是代码评审、diff 和历史清晰，适合工程团队；数据库或平台的优势是动态发布、权限和灰度更方便，适合业务频繁迭代。关键不是存哪里，而是能不能做到版本化、评测门禁、灰度、回滚、审计和线上 trace 关联。

**Q10：Prompt 改动导致 token 成本上涨，你会怎么处理？**

标准答案：我会先从 trace 看输入 token 是否因为系统提示、few-shot 或变量内容变长而上涨，再确认输出长度是否被新 prompt 放宽。处理方式包括压缩系统提示、减少低收益示例、把长规则外置为检索或策略配置、限制变量长度，并在发布门禁里加入 token 成本对比。PromptOps 里成本也应该是质量门禁的一部分。

**Q11：如何处理 prompt 中的用户输入，避免提示注入？**

标准答案：我不会把用户输入直接拼到系统指令里，而是作为明确边界的变量注入，并在模板中标注它是不可信内容。服务端还要做长度限制、脱敏、风险识别和工具权限强校验。即使 prompt 写了“不要听用户覆盖系统指令”，也不能只靠模型自觉，关键权限必须由代码和策略层控制。

**Q12：你会如何设计一个最小可用的 PromptOps 系统？**

标准答案：我会先做五件事：第一，建立 prompt registry，保存 prompt_id、version、owner、场景、风险等级和模板变量；第二，所有调用 trace 记录 prompt_version；第三，接入一套固定评测集和发布门禁；第四，支持按场景或用户 hash 灰度；第五，支持一键回滚到上一组 release 配置。这样即使功能不复杂，也能覆盖面试最关心的可追溯、可验证和可回滚。
