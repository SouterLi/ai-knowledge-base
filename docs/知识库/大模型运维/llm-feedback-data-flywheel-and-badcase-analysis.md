# LLM 应用数据闭环与坏 Case 归因

## 核心概念

### 1. 数据闭环

把线上请求、反馈、审核、日志变成可追踪、可标注、可评估、可复用的资产，驱动 Prompt/RAG/工具/路由/微调持续改进。

```text
线上请求 → 日志/反馈 → 坏 Case 筛选 → 归因 → 标注脱敏 → 评测集/训练集/知识修复 → 离线回归 → 灰度 → 指标追踪
```

目标：每个高价值问题可**复现、定位、修复、验证**——不是数据越多越好。

### 2. 坏 Case 类型

| 类型 | 示例 | 常见根因 |
| --- | --- | --- |
| 事实错误 | 编造政策/价格 | 未召回、过期、幻觉 |
| 拒答错误 | 有答案却说不知道 | Prompt 过严、阈值高 |
| 越权 | 无权限文档内容 | 权限/缓存隔离 |
| 工具错误 | 调错工具/参数 | Schema 不清、缺幂等 |
| 格式错误 | JSON 不可解析 | 约束不足 |
| 成本异常 | 超长上下文 | 裁剪/路由差 |

### 3. 归因维度

输入、知识、检索、上下文、Prompt、模型、工具、产品预期——忌一律归因为「模型不行」。

### 4. 数据资产分层

| 资产 | 用途 |
| --- | --- |
| 线上日志 | 复现链路 |
| 反馈样本 | 发现不满 |
| 坏 Case 池 | 待归因修复 |
| 回归评测集 | 防回退 |
| 训练集 | SFT/DPO（高质量） |
| 知识修复清单 | 文档缺失/过期 |

---

## 核心知识点

### 1. 日志必含字段

request_id、脱敏 user/tenant、输入与轮次、Prompt/模型/索引版本、RAG query 与召回 ID/分数、工具名/参数/耗时、输出与引用、token/延迟/成本、反馈与审核。注意脱敏与保留周期。

### 2. 反馈信号（不只点赞踩）

点踩、重新提问、人工转接、采纳、编辑距离、工单结果、审核拦截——行为≠质量，需抽检与标注校准。

### 3. 坏 Case 处理流程

采样 → 去重 → 脱敏 → 复现 → 归因 → 定级 → 修复 → 回归 → 上线追踪。

### 4. 归因标签示例

```yaml
case_id: "support_20260522_001"
failure_type: "incorrect_answer"
root_cause:
  primary: "retrieval_miss"
  secondary: "query_rewrite_error"
action: { target: "rag", fix: "hybrid_retrieval" }
dataset: { add_to_regression: true, add_to_training: false }
privacy: { pii_removed: true }
```

### 5. 修复优先级（非一律微调）

1. 知识缺失/过期 → 修知识库  
2. 未召回 → 检索/重排/query rewrite  
3. 上下文有答案但不遵循 → Prompt/换模型  
4. 工具不稳 → Schema/校验/示例  
5. 任务稳定且样本足 → SFT/DPO  

### 6. 评测集迭代

纳入高频、高风险、历史坏 Case、对照样本；排除不可复现、未脱敏、重复、规则将变的样本。

### 7. 闭环指标（三层）

- 数据层：处理率、归因覆盖率、标注一致率  
- 质量层：回归通过率、忠实度、工具成功率  
- 业务层：转人工率、满意度、解决率  

### 8. 代码：是否进入回归集

```python
from dataclasses import dataclass
from enum import Enum

class RootCause(str, Enum):
    RETRIEVAL_MISS = "retrieval_miss"
    PROMPT_WEAKNESS = "prompt_weakness"
    TOOL_ERROR = "tool_error"
    KNOWLEDGE_STALE = "knowledge_stale"

@dataclass
class BadCase:
    case_id: str
    root_cause: RootCause
    severity: str
    pii_removed: bool
    reproducible: bool
    duplicated: bool

def should_add_to_regression(c: BadCase) -> bool:
    # 中文注释：回归集要求可复现、已脱敏、非重复
    if not c.pii_removed or not c.reproducible or c.duplicated:
        return False
    return c.severity in {"high", "critical"} or c.root_cause in {
        RootCause.RETRIEVAL_MISS, RootCause.TOOL_ERROR, RootCause.KNOWLEDGE_STALE
    }
```

---

## 高频面试问题与标准答案

**Q1：什么是数据闭环？**  
采集 → 筛选坏 Case → 归因 → 标注脱敏 → 沉淀评测/训练/知识修复 → 回归 → 灰度 → 追踪；核心是复现与验证。

**Q2：错误回答如何排查？**  
用 request_id 拉全链路：检索、上下文、工具、Prompt 版本；判断知识是否存在、是否召回、是否遵循；再选修复路径并加入回归集。

**Q3：坏 Case 都进评测集吗？**  
否。要代表性、稳定、可复现；临时样本先进坏 Case 池。

**Q4：点踩能直接微调吗？**  
不能。点踩无标准答案；需归因、标注、脱敏；检索/知识问题应先修 RAG。

**Q5：归因标签怎么设计？**  
failure_type（现象）+ root_cause（可行动根因）+ 场景/严重级/是否可复现。

**Q6：隐私如何防？**  
最小采集、脱敏、权限、保留周期、租户隔离；进训练/评测前再校验。

**Q7：与 LLMOps 评估区别？**  
评估衡量效果；闭环把线上问题变成改进资产；二者互相供给样本与验证。

**Q8：修 Prompt、RAG 还是微调？**  
按根因：无知识修库；未召回修检索；上下文对但不遵循修 Prompt/模型；大量稳定行为问题再微调。

**Q9：闭环有效性？**  
看三层指标及同类问题是否下降，非单 Case。

**Q10：反馈太多处理不过来？**  
按风险/频率/价值排序；聚类去重；规则/LLM 初筛 + 人工抽高风险。

**Q11：可回放请求？**  
固定 Prompt/模型/检索/工具版本；外部数据变则用快照或 mock。

**Q12：LLM-as-Judge？**  
可用于初筛/聚类/标签建议；高风险需人工与规则，定期校准裁判一致性。

---

## 面试回答加分点

1. **完整答题模板**：目标 → 采集 → 归因 → 分层 → 修复路径 → 治理（脱敏/灰度/回滚）。  
2. **反对**：点踩即训练、只存问答不存链路、万事改 Prompt、评测集只增不删。  
3. **抽象一类问题**：单 Case 修复要上升到规则/检索策略/模板变化。  
4. **与微调关系**：闭环决定「是否该训」；训练集与评测集分开管理。  
5. **生产四词**：可复现、可审计、可回滚、可度量。  
6. **示例口述**：request_id 串联 → 坏 Case 池归因 → 回归 → 灰度看转人工率。
