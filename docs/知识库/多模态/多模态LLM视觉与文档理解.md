# 多模态 LLM：图像、OCR 与文档理解

## 核心概念

### 1. 多模态应用是什么

接收图片、PDF、截图等视觉输入，完成理解、问答、抽取、审核。难点不是「能看图」，而是把非结构化视觉材料变成**可验证、可追溯、可落库**的结果。

### 2. 三条技术路线

| 路线 | 适用 | 优点 | 缺点 |
| --- | --- | --- | --- |
| A. 直接视觉 LLM | 少量图、理解/描述/VQA | 链路短 | 成本高、小字/表格不稳 |
| B. OCR/版面解析 + 文本 LLM | 合同、票据、表单 | 可追溯、易评估、便宜 | OCR 错误传播 |
| C. 混合 | 既要图语义又要准确文字 | 生产常见 | 工程复杂 |

### 3. 面试必强调的四点

1. 输入需预处理（压缩、分页、裁剪、质量检测）。  
2. 输出不可直接信任（Schema、规则、置信度）。  
3. 结论需可追溯（页码、bbox、原文片段）。  
4. 评估分层：OCR、版面、字段、问答、工程指标分开看。

---

## 核心知识点

### 1. 端到端链路

```text
上传 → 类型识别 → 预处理 → OCR/版面 → 分页/裁剪 → 推理 → Schema 校验 → 规则复核 → 落库/溯源 → 人工复核/回流
```

### 2. 预处理

格式转换、分辨率（过低丢小字/过高增 token）、旋转矫正、分页、区域裁剪、质量检测（模糊/缺页则拦截重传）。

### 3. OCR 与版面块

```json
{
  "page": 3,
  "text": "合同金额：人民币 120,000 元",
  "bbox": [120, 380, 520, 420],
  "confidence": 0.96,
  "block_type": "paragraph"
}
```

支撑引用溯源、表格重建、人工复核。

### 4. 表格与长 PDF

表格：先定位区域 → 恢复结构（HTML/JSON）→ 金额/日期校验 → 保留单元格坐标。  
长 PDF：页级 OCR + 检索相关页再精读，避免整本塞入模型。

```python
def answer_pdf_question(question, pages):
  # 中文注释：先用 OCR 文本召回候选页，再对候选页做视觉问答
  candidate_pages = retrieve_pages_by_ocr(question, pages, top_k=5)
  answers = [vision_qa(question, p.image, p.ocr_blocks) for p in candidate_pages]
  return synthesize_with_citations(question, answers)  # 合并须保留页码证据
```

### 5. 结构化输出与校验

```python
from pydantic import BaseModel, Field, field_validator

class InvoiceExtraction(BaseModel):
    invoice_number: str | None
    total_amount: float | None = Field(ge=0)
    source_page: int = Field(ge=1)
    needs_review: bool

    @field_validator("currency")
    @classmethod
    def validate_currency(cls, v):
        # 中文注释：限制枚举，避免模型自由文本无法入库
        if v not in {"CNY", "USD", "EUR"}:
            raise ValueError("unsupported currency")
        return v
```

Prompt 要求：仅基于给定内容、看不清返回 null、关键字段带 evidence。

### 6. 置信度与人工复核

综合 OCR 置信度、规则校验、证据是否齐全；高置信自动通过，低置信强制复核。

### 7. 安全

脱敏、租户隔离、视觉 Prompt Injection（图中文字不可覆盖系统指令）、最小保留原图周期。

---

## 高频面试问题与标准答案

**Q1：多模态 LLM 和 OCR 关系？**  
OCR 提取文字；多模态理解语义与布局。文档类任务 OCR 便宜可追溯；看图语义、签章、缺陷检测偏视觉模型；生产常组合。

**Q2：何时直接用视觉模型？**  
依赖视觉语义（缺陷、截图布局、签章）优先视觉；文字密集抽取优先 OCR+文本模型；复杂场景混合。

**Q3：为何易幻觉？**  
分辨率不足、OCR 错、缺页、Prompt 未要求 null、模型爱补全。治理：质量检测、证据字段、Schema、低置信复核、分层评估。

**Q4：如何可追溯？**  
字段带 page、bbox、source_text；前端可跳转原图区域；落库保存证据链。

**Q5：如何评估合同抽取？**  
按字段算 P/R/F1；关键字段单独看；加 OCR 错误率、引用命中、复核率、P95、单文档成本；覆盖多模板与低质量扫描。

**Q6：表格为何难？**  
合并单元格、多级表头、跨页、单位继承、OCR 顺序乱。先定位表格区再结构恢复，数值校验并保留坐标。

**Q7：长 PDF 怎么处理？**  
页级索引 + 检索相关页精读；固定任务可先识别金额页/签署页；合并答案保留页码。

**Q8：如何控成本延迟？**  
任务路由（OCR+小模型 vs 强视觉）、按页检索、裁剪关键区域、合适分辨率、监控单文档成本。

**Q9：低质量图？**  
质量检测 → 矫正或要求重传；不让模型在不可读图上猜测；低置信标复核。

**Q10：视觉 Prompt Injection？**  
图中文字是**不可信用户输入**，不能覆盖系统指令；工具调用仍要权限校验。

---

## 面试回答加分点

1. **系统设计题结构**：输入类型 → 输出类型 → 预处理 → OCR/视觉组合 → Schema → 校验复核 → 分层指标 → 安全。  
2. **反对「整 PDF 丢给大模型」**：讲清页数、成本、表格、证据链。  
3. **合同/票据方案**：关键页识别、证据页码、签章可能需视觉补充。  
4. **PDF 问答 = 多模态 RAG**：索引含页、图、表；答案带引用。  
5. **JSON 失败**：结构化输出 + Pydantic + 有限重试 + 人工兜底，不用字符串截取入库。  
6. **复核闭环**：展示原图区与 OCR 原文；人工修改回流评估集。  
7. **一分钟总结**：可理解、可校验、可追溯——三句话收束。
