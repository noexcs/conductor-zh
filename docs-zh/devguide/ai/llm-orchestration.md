---
description: 使用 Conductor 进行原生 LLM 编排 — 支持的 LLM 提供方、用于 RAG 管道的向量数据库集成，以及多模态内容生成任务。
---

# LLM 编排

Conductor 为 LLM 编排与集成提供原生系统任务。不需要外部框架或自定义工作者 — 配置一个提供方，就能在任何工作流中使用它。每个提供方都通过 MCP 工具集成支持函数调用（function calling）。

## 支持的 LLM 提供方 { #supported-llm-providers }

| 提供方 | 对话补全 | 文本补全 | 嵌入 |
|---|---|---|---|
| Anthropic (Claude) | ✓ | ✓ | — |
| OpenAI (GPT) | ✓ | ✓ | ✓ |
| Azure OpenAI | ✓ | ✓ | ✓ |
| Google Gemini | ✓ | ✓ | ✓ |
| AWS Bedrock | ✓ | ✓ | ✓ |
| Mistral | ✓ | ✓ | ✓ |
| Cohere | ✓ | ✓ | ✓ |
| HuggingFace | ✓ | ✓ | ✓ |
| Ollama | ✓ | ✓ | ✓ |
| Perplexity | ✓ | — | — |
| Grok (xAI) | ✓ | ✓ | — |
| StabilityAI | — | — | — |

每个提供方都在任务上配置，因此工作流可以为该步骤选择合适的模型，而不必改动周围的编排逻辑。


## 内置工具与高级能力

Conductor 支持提供方原生的工具，它们运行在提供方的基础设施上 — 不需要 MCP 服务器或自定义工作者。在 `LLM_CHAT_COMPLETE` 任务中用一个参数即可启用它们。

| 能力 | 参数 | OpenAI | Anthropic | Google Gemini |
|---|---|---|---|---|
| Web 搜索 | `webSearch: true` | ✓ | ✓ | ✓ |
| 代码执行 | `codeInterpreter: true` | ✓ (code_interpreter) | ✓ (code_execution) | ✓ (code_execution) |
| 文件搜索 | `fileSearchVectorStoreIds: [...]` | ✓ | — | — |
| 扩展思考 | `thinkingTokenLimit: N` | — | ✓ | ✓ |
| 推理强度 | `reasoningEffort: "high"` | ✓ | — | — |
| Google 搜索 | `googleSearchRetrieval: true` | — | — | ✓ |
| 自定义函数 | `tools: [...]` | ✓ | ✓ | ✓ |

### Web 搜索

LLM 可以在对话补全期间搜索网络以获取实时信息。使用 `"webSearch": true` 启用：

```json
{
  "type": "LLM_CHAT_COMPLETE",
  "inputParameters": {
    "llmProvider": "openai",
    "model": "gpt-4o-mini",
    "messages": [{"role": "user", "message": "What happened in tech news today?"}],
    "webSearch": true
  }
}
```

适用于 OpenAI、Anthropic 和 Google Gemini。每个提供方使用各自的网络搜索原生实现。

### 代码执行

LLM 可以在沙箱环境中编写并执行代码。使用 `"codeInterpreter": true` 启用：

```json
{
  "type": "LLM_CHAT_COMPLETE",
  "inputParameters": {
    "llmProvider": "google_gemini",
    "model": "gemini-2.5-flash",
    "messages": [{"role": "user", "message": "Calculate the first 100 prime numbers and plot them"}],
    "codeInterpreter": true
  }
}
```

将其用于数据分析、图表生成、数学计算，或任何受益于运行代码的任务。

### 扩展思考

在 LLM 响应之前，为它提供一个用于逐步推理的 token 预算。适用于受益于思维链（chain-of-thought）推理的复杂问题：

```json
{
  "type": "LLM_CHAT_COMPLETE",
  "inputParameters": {
    "llmProvider": "anthropic",
    "model": "claude-sonnet-4-20250514",
    "messages": [{"role": "user", "message": "Prove that there are infinitely many primes"}],
    "thinkingTokenLimit": 10000,
    "maxTokens": 16000
  }
}
```

Anthropic 和 Google Gemini 支持此能力。


## 向量数据库工作流

内置的向量数据库集成让 RAG（检索增强生成）管道成为标准的向量数据库工作流。

| 向量数据库 | 存储嵌入 | 索引文本 | 语义搜索 |
|---|---|---|---|
| Pinecone | ✓ | ✓ | ✓ |
| pgvector (PostgreSQL) | ✓ | ✓ | ✓ |
| MongoDB Atlas Vector Search | ✓ | ✓ | ✓ |


### 示例：RAG 管道

一个使用原生系统任务的完整 RAG 工作流 — 索引文档、搜索，并生成回答。不需要自定义工作者。

```json
{
  "name": "rag_pipeline",
  "description": "Index documents, search, and generate RAG answer",
  "version": 1,
  "schemaVersion": 2,
  "tasks": [
    {
      "name": "index_document",
      "taskReferenceName": "index_ref",
      "type": "LLM_INDEX_TEXT",
      "inputParameters": {
        "vectorDB": "postgres-prod",
        "index": "knowledge_base",
        "namespace": "docs",
        "docId": "${workflow.input.docId}",
        "text": "${workflow.input.text}",
        "embeddingModelProvider": "openai",
        "embeddingModel": "text-embedding-3-small",
        "dimensions": 1536,
        "metadata": "${workflow.input.metadata}"
      }
    },
    {
      "name": "search_index",
      "taskReferenceName": "search_ref",
      "type": "LLM_SEARCH_INDEX",
      "inputParameters": {
        "vectorDB": "postgres-prod",
        "index": "knowledge_base",
        "namespace": "docs",
        "query": "${workflow.input.question}",
        "embeddingModelProvider": "openai",
        "embeddingModel": "text-embedding-3-small",
        "dimensions": 1536,
        "maxResults": 3
      }
    },
    {
      "name": "generate_answer",
      "taskReferenceName": "answer_ref",
      "type": "LLM_CHAT_COMPLETE",
      "inputParameters": {
        "llmProvider": "openai",
        "model": "gpt-4o-mini",
        "messages": [
          {
            "role": "system",
            "message": "Answer the question using only the provided context."
          },
          {
            "role": "user",
            "message": "Context:\n${search_ref.output.result}\n\nQuestion: ${workflow.input.question}"
          }
        ],
        "temperature": 0.2
      }
    }
  ],
  "outputParameters": {
    "searchResults": "${search_ref.output.result}",
    "answer": "${answer_ref.output.result}"
  }
}
```

每一种任务类型 — `LLM_INDEX_TEXT`、`LLM_SEARCH_INDEX`、`LLM_CHAT_COMPLETE` — 都是原生 Conductor 系统任务。向量数据库、嵌入模型和 LLM 提供方都是配置参数。把一个参数值改掉，就能从 pgvector 切换到 Pinecone，或从 OpenAI 切换到 Anthropic。


## 内容生成

用于多模态内容生成的原生系统任务：

| 任务 | 类型 | 描述 |
|---|---|---|
| 生成图像 | `GENERATE_IMAGE` | 通过 AI 模型进行文生图 |
| 生成音频 | `GENERATE_AUDIO` | 文本转语音合成 |
| 生成视频 | `GENERATE_VIDEO` | 文本/图像转视频生成（异步） |
| 生成 PDF | `GENERATE_PDF` | Markdown 转 PDF 文档转换 |


## 示例

覆盖每一种 AI 任务类型的即用型工作流定义。每个示例都是一个完整的工作流 JSON，你可以直接注册并运行。

| 示例 | 使用的任务类型 |
|---|---|
| [对话补全](https://github.com/conductor-oss/conductor/blob/main/ai/examples/01-chat-completion.json) | `LLM_CHAT_COMPLETE` |
| [生成嵌入](https://github.com/conductor-oss/conductor/blob/main/ai/examples/02-generate-embeddings.json) | `LLM_GENERATE_EMBEDDINGS` |
| [图像生成](https://github.com/conductor-oss/conductor/blob/main/ai/examples/03-image-generation.json) | `GENERATE_IMAGE` |
| [音频生成](https://github.com/conductor-oss/conductor/blob/main/ai/examples/04-audio-generation.json) | `GENERATE_AUDIO` |
| [语义搜索](https://github.com/conductor-oss/conductor/blob/main/ai/examples/05-semantic-search.json) | `LLM_SEARCH_INDEX` |
| [RAG 基础](https://github.com/conductor-oss/conductor/blob/main/ai/examples/06-rag-basic.json) | `LLM_SEARCH_INDEX`, `LLM_CHAT_COMPLETE` |
| [RAG 完整](https://github.com/conductor-oss/conductor/blob/main/ai/examples/07-rag-complete.json) | `LLM_INDEX_TEXT`, `LLM_SEARCH_INDEX`, `LLM_CHAT_COMPLETE` |
| [MCP 列出工具](https://github.com/conductor-oss/conductor/blob/main/ai/examples/08-mcp-list-tools.json) | `LIST_MCP_TOOLS` |
| [MCP 调用工具](https://github.com/conductor-oss/conductor/blob/main/ai/examples/09-mcp-call-tool.json) | `CALL_MCP_TOOL` |
| [MCP AI Agent](https://github.com/conductor-oss/conductor/blob/main/ai/examples/10-mcp-ai-agent.json) | `LIST_MCP_TOOLS`, `LLM_CHAT_COMPLETE`, `CALL_MCP_TOOL` |
| [视频 — OpenAI Sora](https://github.com/conductor-oss/conductor/blob/main/ai/examples/11-video-openai-sora.json) | `GENERATE_VIDEO` |
| [视频 — Gemini Veo](https://github.com/conductor-oss/conductor/blob/main/ai/examples/12-video-gemini-veo.json) | `GENERATE_VIDEO` |
| [图生视频管道](https://github.com/conductor-oss/conductor/blob/main/ai/examples/13-image-to-video-pipeline.json) | `GENERATE_IMAGE`, `GENERATE_VIDEO` |
| [StabilityAI 图像](https://github.com/conductor-oss/conductor/blob/main/ai/examples/14-stabilityai-image.json) | `GENERATE_IMAGE` |
| [PDF 生成](https://github.com/conductor-oss/conductor/blob/main/ai/examples/15-pdf-generation.json) | `GENERATE_PDF` |
| [LLM 转 PDF 管道](https://github.com/conductor-oss/conductor/blob/main/ai/examples/16-llm-to-pdf-pipeline.json) | `LLM_CHAT_COMPLETE`, `GENERATE_PDF` |
| [Web 搜索](https://github.com/conductor-oss/conductor/blob/main/ai/examples/17-web-search.json) | `LLM_CHAT_COMPLETE` (web 搜索) |
| [代码执行](https://github.com/conductor-oss/conductor/blob/main/ai/examples/18-code-execution.json) | `LLM_CHAT_COMPLETE` (代码执行) |
| [编程 Agent](https://github.com/conductor-oss/conductor/blob/main/ai/examples/19-coding-agent.json) | `LLM_CHAT_COMPLETE` (code_interpreter) |
| [扩展思考](https://github.com/conductor-oss/conductor/blob/main/ai/examples/20-extended-thinking.json) | `LLM_CHAT_COMPLETE` (thinking) |
| [Web 研究 Agent](https://github.com/conductor-oss/conductor/blob/main/ai/examples/21-web-search-research-agent.json) | `LLM_CHAT_COMPLETE` (web 搜索 + thinking), `GENERATE_PDF` |
| [多轮对话链](https://github.com/conductor-oss/conductor/blob/main/ai/examples/22-multi-turn-chain.json) | `LLM_CHAT_COMPLETE` (previousResponseId) |
| [带 AI 的动态工作流](https://github.com/conductor-oss/conductor/blob/main/ai/examples/36-ai-workflow-routing.json) | `LLM_CHAT_COMPLETE`, 动态 `SUB_WORKFLOW` |

浏览所有示例：[`ai/examples/`](https://github.com/conductor-oss/conductor/tree/main/ai/examples)


## 后续步骤

- **[生产级 Agent 架构](production-agent-architecture.md)** &mdash; 在这些任务周围添加治理、评估、部署、恢复和运维。
- **[持久化自适应图](dynamic-workflows.md)** &mdash; 治理运行时生成的计划、有界扇出、审批与恢复。
- **[生产级 Agent 架构](production-agent-architecture.md)** &mdash; 把 LLM 编排放进持久、可观测的生产边界。
- **[动态工作流](dynamic-workflows.md)** &mdash; 在运行时自行构建执行计划的 agent。
- **[AI Cookbook](cookbook/index.md)** &mdash; 常见 LLM、工具和 agent 工作流模式的生产级起步模板。
