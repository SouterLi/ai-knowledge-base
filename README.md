# AI 应用开发面试笔记

本仓库收录 AI 应用开发相关面试主题的整理笔记，按主题分类存放于 `docs/interview/ai-application-development/` 目录下。

## 目录结构

| 主题分类 | 路径 | 文档 |
| --- | --- | --- |
| Agent | `agents/` | [工具调用与函数调用设计](docs/interview/ai-application-development/agents/agent-tool-calling-and-function-design.md)、[规划执行与可靠性治理](docs/interview/ai-application-development/agents/ai-agent-planning-execution-and-reliability.md) |
| RAG | `rag/` | [系统设计与评估](docs/interview/ai-application-development/rag/rag-system-design-and-evaluation.md)、[Embedding 与向量索引工程](docs/interview/ai-application-development/rag/embedding-vector-index-engineering.md)、[GraphRAG 与知识图谱增强检索](docs/interview/ai-application-development/rag/graphrag-knowledge-graph-retrieval.md) |
| 上下文工程 | `context-engineering/` | [上下文与记忆设计](docs/interview/ai-application-development/context-engineering/llm-context-memory-design.md) |
| 数据查询 | `data-query/` | [Text-to-SQL / NL2SQL 应用开发](docs/interview/ai-application-development/data-query/text-to-sql-nl2sql-application-development.md) |
| Prompt 工程 | `prompt-engineering/` | [Prompt 工程与结构化输出](docs/interview/ai-application-development/prompt-engineering/prompt-engineering-and-structured-output.md) |
| 模型适配与微调 | `fine-tuning/` | [LLM 微调与 PEFT 落地](docs/interview/ai-application-development/fine-tuning/llm-fine-tuning-and-peft.md) |
| 多模态 | `multimodal/` | [多模态 LLM 图像与文档理解](docs/interview/ai-application-development/multimodal/multimodal-llm-vision-document-understanding.md) |
| MCP | `mcp/` | [MCP 服务端与客户端集成](docs/interview/ai-application-development/mcp/model-context-protocol-server-and-client-integration.md) |
| 运行时架构 | `runtime/` | [流式响应与异步任务架构](docs/interview/ai-application-development/runtime/llm-streaming-and-async-architecture.md) |
| 测试工程 | `testing/` | [LLM 应用测试与 Mock 策略](docs/interview/ai-application-development/testing/llm-application-testing-and-mocking.md) |
| Workflow | `workflow/` | [LLM 工作流编排与 Human-in-the-loop](docs/interview/ai-application-development/workflow/llm-workflow-orchestration-and-human-in-the-loop.md) |
| 生产化 | `production/` | [成本、缓存与限流设计](docs/interview/ai-application-development/production/llm-cost-cache-rate-limit.md)、[模型网关与多模型路由](docs/interview/ai-application-development/production/llm-model-gateway-and-routing.md)、[发布、配置与实验治理](docs/interview/ai-application-development/production/llm-release-config-and-experiment-governance.md) |
| LLMOps | `llmops/` | [评估与可观测性](docs/interview/ai-application-development/llmops/llm-application-evaluation-observability.md)、[数据闭环与坏 Case 归因](docs/interview/ai-application-development/llmops/llm-feedback-data-flywheel-and-badcase-analysis.md) |
| 安全 | `security/` | [应用安全防护](docs/interview/ai-application-development/security/llm-application-security.md) |

## 文件树

```
docs/interview/ai-application-development/
├── agents/
├── context-engineering/
├── data-query/
├── fine-tuning/
├── llmops/
├── mcp/
├── multimodal/
├── production/
├── prompt-engineering/
├── rag/
├── runtime/
├── security/
├── testing/
└── workflow/
```
