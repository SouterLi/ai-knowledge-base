# LLM 应用发布、配置与实验治理

## 核心概念

### 1. 行为配置

不改业务代码但会改变模型输出的配置：**Prompt 模板、模型别名、生成参数、temperature/max_tokens、RAG top_k/阈值、工具 schema、安全策略**。它们都应进入与代码同级的发布流程（审批、灰度、审计、回滚）。

### 2. Prompt 版本

不仅是模板文本，而是包含变量契约、依赖模型、评测结果、发布状态、回滚目标的**快照**。

```yaml
id: refund_prompt_v12
task: refund_policy_answer
model_alias: chat_quality
status: canary
eval_report:
  pass_rate: 0.94
  format_valid_rate: 0.99
```

### 3. 灰度发布

先少量流量观察**质量、格式、安全、工具成功率、成本、P95**，再扩量。LLM 灰度不能只看 HTTP 5xx——可能“成功但答错”。

### 4. A/B 实验

对比 Prompt、模型、RAG 策略等；要求**稳定分流**（同用户/会话不频繁切组）、**单变量**、**完整版本日志**。

### 5. 回滚

回滚对象常是配置而非代码：Prompt 版本、模型映射、RAG 参数、关闭工具、收紧安全策略。目标：**分钟级配置切回，无需重新部署**。

---

## 核心知识点

### 1. 推荐架构

```text
请求 → 实验分流 → 配置解析 → Prompt/模型/RAG/工具选择 → 编排执行
     → 输出校验 → 指标采集 → 实验分析/回滚
```

| 模块 | 职责 |
| --- | --- |
| 配置中心 | 存 Prompt、参数、RAG、工具、安全策略 |
| 发布控制面 | 草稿→评测→灰度→全量→暂停→回滚 |
| 运行时解析器 | 解析当前请求命中的配置版本 |

### 2. 配置分层

```text
全局默认 → 业务线 → 场景 → 租户覆盖 → 实验覆盖 → 紧急开关（最高优先级）
```

```python
from dataclasses import dataclass

@dataclass
class RuntimeConfig:
  prompt_version: str
  model_alias: str
  temperature: float
  rag_top_k: int
  tool_calling_enabled: bool

def resolve_config(base: dict, tenant_id: str, variant: str | None) -> RuntimeConfig:
  cfg = dict(base["default"])
  if tenant_id in base.get("tenant_overrides", {}):
    cfg.update(base["tenant_overrides"][tenant_id])
  if variant == "candidate":
    cfg.update(base["experiments"]["prompt_ab"]["variant"])
  if base.get("kill_switch", {}).get("disable_tool_calling"):
    cfg["tool_calling_enabled"] = False  # 中文注释：紧急开关用于事故止血
  return RuntimeConfig(**{k: cfg[k] for k in RuntimeConfig.__dataclass_fields__})
```

### 3. 发布流程

草稿 → 静态校验（变量/schema）→ 离线评测（黄金样例+安全样例）→ 审批 → 小流量灰度 → 指标观察 → 扩量/回滚。可称为 **Prompt 的 CI/CD**。

### 4. 上线门禁示例

```text
golden_pass_rate < 0.90 → 禁止灰度
pii_leak > 0 → 禁止上线
output_format_valid < 0.98 → 禁止全量
p95_latency 增长 > 30% → 人工审批
```

### 5. 稳定分流

```python
import hashlib

def choose_variant(exp_id: str, subject_id: str, traffic: float) -> str:
  key = f"{exp_id}:{subject_id}".encode()
  bucket = int(hashlib.sha256(key).hexdigest()[:8], 16) / 0xFFFFFFFF
  # 中文注释：同一 subject 在同一实验中持续命中同一组
  return "candidate" if bucket < traffic else "control"
```

分流键：用户体验用 `user_id`，多轮对话用 `conversation_id`，租户隔离用 `tenant_id`。

### 6. 运行时版本日志

每次调用记录：experiment_id、variant、prompt_version、model_alias、resolved_model、rag_policy_version、safety_policy_version。无此字段无法定位坏 Case。

### 7. 与模型网关边界

- **网关**：怎么调用（Adapter、重试、限流、计费）。
- **发布治理**：谁用什么行为版本（Prompt、别名、参数）。
- 实验逻辑不要全部塞进网关，避免变成“策略大杂烩”。

---

## 高频面试问题与标准答案

**Q1：Prompt 为什么需要版本管理？**

Prompt 是生产逻辑；无版本则无法定位哪次改动导致问题，也无法复现实验。版本应含模板、变量 schema、模型依赖、评测结果、状态与回滚目标。

**Q2：LLM 灰度与普通后端灰度有何不同？**

后端主要看错误率/延迟；LLM 还要看输出质量、事实一致性、格式稳定、安全拦截、工具成功率、成本。需离线评测 + 线上质量指标 + 人工抽检。

**Q3：如何设计 Prompt A/B？**

定目标指标 → 稳定分流 → 单变量 → 日志记 experiment/variant/版本 → 发布前离线评测，高风险小流量。

**Q4：Prompt 改坏了如何止损？**

关闭实验或切回旧版本；查日志中 prompt_version/model_alias；安全事故启用 kill_switch（禁工具、收紧安全）；坏 Case 入回归集。

**Q5：哪些变更需人工审批？**

新增外部写工具、放宽安全策略、未评测模型上线、金融/医疗/法务 Prompt、扩大数据访问范围。

**Q6：如何避免实验污染？**

稳定哈希分流；缓存 key 含实验版本；扩流注意样本污染；一次只改一个主要变量。

**Q7：配置中心与代码仓库如何分工？**

稳定编排/解析/校验放代码；高频 Prompt/参数/RAG 放配置中心（版本+审批+灰度+审计）；高风险可用 GitOps PR 审核后同步。

**Q8：何时直接回滚不等显著性？**

安全泄露、越权、核心格式大面积失败、关键任务失败率飙升、成本失控、投诉激增——高风险场景先止血。

---

## 面试回答加分点

1. 开场区分**代码变更**与**行为配置变更**，体现 LLM 特有发布观。
2. 配置解析顺序**固定且可解释**（默认→租户→实验→kill_switch）。
3. 强调**日志带全版本**是定位与回滚的前提。
4. 灰度指标用**多目标**表述：质量不能以安全失控为代价。
5. 回滚**不依赖发版**，体现平台化成熟度。
6. 坏 Case **回流评测集**，形成闭环。
7. 口诀：**版本先行，评测把关；稳定分流，指标护航；日志可追，配置可回**。
