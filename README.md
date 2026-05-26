# AI 应用开发面试笔记

> 一本面向 **AI 应用开发 / LLM 应用 / Agent·RAG 工程师** 岗位的面试复习手册。  
> 正文按主题拆分为独立篇章，每篇围绕「核心概念 → 核心知识点 → 高频面试题与参考答案」组织，便于按需精读或考前速览。

---

## 关于本书

企业里的大模型应用，面试很少只问「API 怎么调」，更常追问 **能不能落地、能不能稳住、能不能量化改进**。本仓库把这类问题拆成可复习的专题：从 Prompt、RAG、Agent 到生产化、安全与 LLMOps，尽量用工程视角写清楚「设计取舍」和「面试怎么答」，而不是概念罗列。

**适合谁读**

- 准备 AI 应用开发、LLM 平台、智能客服 / 知识库、Agent 方向面试的工程师  
- 已在写 RAG / Agent 产品，希望系统化补全知识盲区的开发者  

**怎么读**

- **冲刺复习**：优先阅读 **第 1～12 篇**（覆盖绝大多数一面、二面高频点）  
- **岗位定制**：知识库 / 问答岗建议 2、5～7、22；Agent 岗建议 3、8、13、16、21；平台 / 基建岗建议 10～12、14、17～20、27  
- **专题深入**：同一主题多篇（如 RAG 共五篇）建议先读第 2 篇总览，再读第 5～7、22 篇  

所有正文位于 `docs/interview/ai-application-development/`。

---

## 目录

全书 **27 篇**，下表按 **面试常见程度、岗位通用性、知识依赖关系** 排序。序号越小，越建议优先阅读；「卷」仅作主题归类，**复习顺序以序号为准**。

| 篇 | 卷 | 主题 | 篇章 |
| ---: | --- | --- | --- |
| 1 | 基础能力 | Prompt 工程与结构化输出 | [阅读](docs/interview/ai-application-development/prompt-engineering/prompt-engineering-and-structured-output.md) |
| 2 | 基础能力 | RAG 系统设计与评估（总览） | [阅读](docs/interview/ai-application-development/rag/rag-system-design-and-evaluation.md) |
| 3 | 基础能力 | AI Agent 工具调用与 Function Calling 设计 | [阅读](docs/interview/ai-application-development/agents/agent-tool-calling-and-function-design.md) |
| 4 | 基础能力 | LLM 上下文工程与记忆设计 | [阅读](docs/interview/ai-application-development/context-engineering/llm-context-memory-design.md) |
| 5 | RAG 工程化 | 文档摄取、切分与索引构建流水线 | [阅读](docs/interview/ai-application-development/rag/rag-document-ingestion-and-chunking.md) |
| 6 | RAG 工程化 | Embedding 与向量索引工程 | [阅读](docs/interview/ai-application-development/rag/embedding-vector-index-engineering.md) |
| 7 | RAG 工程化 | 重排序与上下文压缩 | [阅读](docs/interview/ai-application-development/rag/rag-reranking-and-context-compression.md) |
| 8 | Agent 与编排 | AI Agent 规划执行与可靠性治理 | [阅读](docs/interview/ai-application-development/agents/ai-agent-planning-execution-and-reliability.md) |
| 9 | 安全与合规 | LLM 应用安全防护 | [阅读](docs/interview/ai-application-development/security/llm-application-security.md) |
| 10 | 生产化与 LLMOps | LLM 应用评估与可观测性 | [阅读](docs/interview/ai-application-development/llmops/llm-application-evaluation-observability.md) |
| 11 | 生产化与 LLMOps | LLM 应用成本、缓存与限流设计 | [阅读](docs/interview/ai-application-development/production/llm-cost-cache-rate-limit.md) |
| 12 | 运行时与性能 | LLM 流式响应与异步任务架构 | [阅读](docs/interview/ai-application-development/runtime/llm-streaming-and-async-architecture.md) |
| 13 | Agent 与编排 | LLM 工作流编排与 Human-in-the-loop | [阅读](docs/interview/ai-application-development/workflow/llm-workflow-orchestration-and-human-in-the-loop.md) |
| 14 | 生产化与 LLMOps | 模型网关与多模型路由 | [阅读](docs/interview/ai-application-development/production/llm-model-gateway-and-routing.md) |
| 15 | 数据与查询 | Text-to-SQL / NL2SQL 应用开发 | [阅读](docs/interview/ai-application-development/data-query/text-to-sql-nl2sql-application-development.md) |
| 16 | Agent 与编排 | MCP 服务端与客户端集成 | [阅读](docs/interview/ai-application-development/mcp/model-context-protocol-server-and-client-integration.md) |
| 17 | 生产化与 LLMOps | LLM 应用测试与 Mock 策略 | [阅读](docs/interview/ai-application-development/testing/llm-application-testing-and-mocking.md) |
| 18 | 安全与合规 | 身份权限与多租户隔离 | [阅读](docs/interview/ai-application-development/security/llm-identity-permission-and-multitenancy.md) |
| 19 | 生产化与 LLMOps | 发布、配置与实验治理 | [阅读](docs/interview/ai-application-development/production/llm-release-config-and-experiment-governance.md) |
| 20 | 生产化与 LLMOps | 数据闭环与坏 Case 归因 | [阅读](docs/interview/ai-application-development/llmops/llm-feedback-data-flywheel-and-badcase-analysis.md) |
| 21 | Agent 与编排 | 多 Agent 协作与编排设计 | [阅读](docs/interview/ai-application-development/agents/multi-agent-collaboration-and-orchestration.md) |
| 22 | RAG 工程化 | GraphRAG 与知识图谱增强检索 | [阅读](docs/interview/ai-application-development/rag/graphrag-knowledge-graph-retrieval.md) |
| 23 | 运行时与性能 | LLM 应用性能与端到端延迟优化 | [阅读](docs/interview/ai-application-development/performance/llm-application-latency-optimization.md) |
| 24 | 专项进阶 | LLM 微调与 PEFT 落地 | [阅读](docs/interview/ai-application-development/fine-tuning/llm-fine-tuning-and-peft.md) |
| 25 | 专项进阶 | 多模态 LLM 图像与文档理解 | [阅读](docs/interview/ai-application-development/multimodal/multimodal-llm-vision-document-understanding.md) |
| 26 | 专项进阶 | 实时语音 Agent 架构设计 | [阅读](docs/interview/ai-application-development/voice-agent/realtime-voice-agent-architecture.md) |
| 27 | 生产化与 LLMOps | 开源大模型私有化部署与推理服务 | [阅读](docs/interview/ai-application-development/deployment/open-source-llm-private-deployment-and-inference-serving.md) |

**卷次说明**

| 卷 | 涵盖篇次 | 面试侧重 |
| --- | --- | --- |
| 基础能力 | 1～4 | Prompt、RAG 总览、Agent 工具、上下文——几乎所有岗位必问 |
| RAG 工程化 | 2、5～7、22 | 切分、索引、重排、图谱——知识库 / 问答岗深度考点 |
| Agent 与编排 | 3、8、13、16、21 | 可靠性、工作流、MCP、多 Agent——Agent 方向核心 |
| 安全与合规 | 9、18 | 提示注入、越权、多租户——企业落地必谈 |
| 生产化与 LLMOps | 10、11、14、17、19、20、27 | 评估、成本、网关、测试、发布、数据闭环与私有化推理服务 |
| 运行时与性能 | 12、23 | 流式、异步、延迟——体验与 SLA |
| 数据与查询 | 15 | NL2SQL——数据分析 / 业务系统结合场景 |
| 专项进阶 | 24～26 | 微调、多模态、语音——按岗位选读 |

---

## 贡献与迭代

新增篇章时请保持与现有正文相同的结构（核心概念、核心知识点、高频面试题及参考答案），并同步更新本目录的篇次、卷次与排序说明。
