---
description: AI Cookbook 已迁移到专属配方库。
redirect_to: devguide/ai/cookbook/index.html
---

# AI 与 LLM 编排配方

使用 Conductor 的原生 AI 能力构建持久化智能体和 LLM 工作流。以下每个配方都在完整的持久化执行保障下运行——重试、状态持久化和崩溃恢复。

### 聊天补全

一个单步工作流，向 LLM 发送问题并返回答案。

```json
{
  "name": "chat_workflow",
  "version": 1,
  "schemaVersion": 2,
  "tasks": [
    {
      "name": "chat_task",
      "taskReferenceName": "chat",
      "type": "LLM_CHAT_COMPLETE",
      "inputParameters": {
        "llmProvider": "openai",
        "model": "gpt-4o-mini",
        "messages": [
          {"role": "system", "message": "You are a helpful assistant."},
          {"role": "user", "message": "${workflow.input.question}"}
        ],
        "temperature": 0.7,
        "maxTokens": 500
      }
    }
  ],
  "inputParameters": ["question"],
  "outputParameters": {
    "answer": "${chat.output.result}"
  }
}
```

**注册并运行：**

```shell
curl -X POST 'http://localhost:8080/api/metadata/workflow' \
  -H 'Content-Type: application/json' \
  -d @chat_workflow.json

curl -X POST 'http://localhost:8080/api/workflow/chat_workflow' \
  -H 'Content-Type: application/json' \
  -d '{"question": "What is workflow orchestration?"}'
```

---

### 基于向量数据库的 RAG 流水线（检索 + 问答）

用于检索增强生成（RAG）的向量数据库工作流：向量检索先获取相关文档，然后由 LLM 基于这些结果生成答案。

```json
{
  "name": "rag_workflow",
  "version": 1,
  "schemaVersion": 2,
  "inputParameters": ["question"],
  "tasks": [
    {
      "name": "search_knowledge_base",
      "taskReferenceName": "search",
      "type": "LLM_SEARCH_INDEX",
      "inputParameters": {
        "vectorDB": "postgres-prod",
        "namespace": "kb",
        "index": "articles",
        "embeddingModelProvider": "openai",
        "embeddingModel": "text-embedding-3-small",
        "query": "${workflow.input.question}",
        "llmMaxResults": 3
      }
    },
    {
      "name": "generate_answer",
      "taskReferenceName": "answer",
      "type": "LLM_CHAT_COMPLETE",
      "inputParameters": {
        "llmProvider": "anthropic",
        "model": "claude-sonnet-4-20250514",
        "messages": [
          {"role": "system", "message": "Answer based on the following context: ${search.output.result}"},
          {"role": "user", "message": "${workflow.input.question}"}
        ],
        "temperature": 0.3
      }
    }
  ],
  "outputParameters": {
    "answer": "${answer.output.result}",
    "sources": "${search.output.result}"
  }
}
```

**注册并运行：**

```shell
curl -X POST 'http://localhost:8080/api/metadata/workflow' \
  -H 'Content-Type: application/json' \
  -d @rag_workflow.json

curl -X POST 'http://localhost:8080/api/workflow/rag_workflow' \
  -H 'Content-Type: application/json' \
  -d '{"question": "How do I configure retry policies?"}'
```

!!! note "前置条件"
    需要一个已配置为 Conductor 集成的向量数据库（pgvector、Pinecone 或 MongoDB Atlas），以及至少一个 LLM 提供商。见下文[AI 提供商配置](#ai-provider-configuration)。

---

### 支持函数调用的 MCP AI 智能体

一个四步的 Agentic 工作流，演示带函数调用的 AI 智能体编排：通过 MCP 发现可用工具，让 LLM 选择合适的工具，通过工具使用执行它，并总结结果。

```json
{
  "name": "mcp_ai_agent_workflow",
  "version": 1,
  "schemaVersion": 2,
  "inputParameters": ["task"],
  "tasks": [
    {
      "name": "list_available_tools",
      "taskReferenceName": "discover_tools",
      "type": "LIST_MCP_TOOLS",
      "inputParameters": {
        "mcpServer": "http://localhost:3001/mcp"
      }
    },
    {
      "name": "decide_which_tools_to_use",
      "taskReferenceName": "plan",
      "type": "LLM_CHAT_COMPLETE",
      "inputParameters": {
        "llmProvider": "anthropic",
        "model": "claude-sonnet-4-20250514",
        "messages": [
          {"role": "system", "message": "You are an AI agent. Available tools: ${discover_tools.output.tools}. User wants to: ${workflow.input.task}"},
          {"role": "user", "message": "Which tool should I use and what parameters? Respond with JSON: {method: string, arguments: object}"}
        ],
        "temperature": 0.1,
        "maxTokens": 500
      }
    },
    {
      "name": "execute_tool",
      "taskReferenceName": "execute",
      "type": "CALL_MCP_TOOL",
      "inputParameters": {
        "mcpServer": "http://localhost:3001/mcp",
        "method": "${plan.output.result.method}",
        "arguments": "${plan.output.result.arguments}"
      }
    },
    {
      "name": "summarize_result",
      "taskReferenceName": "summarize",
      "type": "LLM_CHAT_COMPLETE",
      "inputParameters": {
        "llmProvider": "openai",
        "model": "gpt-4o-mini",
        "messages": [
          {"role": "user", "message": "Summarize this result for the user: ${execute.output.content}"}
        ],
        "maxTokens": 200
      }
    }
  ],
  "outputParameters": {
    "summary": "${summarize.output.result}",
    "rawToolOutput": "${execute.output.content}"
  }
}
```

**注册并运行：**

```shell
curl -X POST 'http://localhost:8080/api/metadata/workflow' \
  -H 'Content-Type: application/json' \
  -d @mcp_ai_agent_workflow.json

curl -X POST 'http://localhost:8080/api/workflow/mcp_ai_agent_workflow' \
  -H 'Content-Type: application/json' \
  -d '{"task": "Look up the latest order status for customer 42"}'
```

---

### 图像生成

使用 gpt-image-1 或其他受支持的提供商，根据文本提示生成图像。

```json
{
  "name": "image_gen_workflow",
  "version": 1,
  "schemaVersion": 2,
  "inputParameters": ["prompt"],
  "tasks": [
    {
      "name": "generate_image",
      "taskReferenceName": "image",
      "type": "GENERATE_IMAGE",
      "inputParameters": {
        "llmProvider": "openai",
        "model": "gpt-image-1",
        "prompt": "${workflow.input.prompt}",
        "width": 1024,
        "height": 1024,
        "n": 1,
        "style": "vivid"
      }
    }
  ],
  "outputParameters": {
    "imageUrl": "${image.output.result}"
  }
}
```

**注册并运行：**

```shell
curl -X POST 'http://localhost:8080/api/metadata/workflow' \
  -H 'Content-Type: application/json' \
  -d @image_gen_workflow.json

curl -X POST 'http://localhost:8080/api/workflow/image_gen_workflow' \
  -H 'Content-Type: application/json' \
  -d '{"prompt": "A futuristic city skyline at sunset, digital art"}'
```

---

### LLM 报告转 PDF 流水线

由 LLM 生成结构化的 markdown 报告，然后 Conductor 将其转换为可下载的 PDF。

```json
{
  "name": "llm_to_pdf_pipeline",
  "description": "LLM generates a markdown report, then converts it to PDF",
  "version": 1,
  "schemaVersion": 2,
  "inputParameters": ["topic", "audience"],
  "tasks": [
    {
      "name": "generate_report_markdown",
      "taskReferenceName": "llm_report",
      "type": "LLM_CHAT_COMPLETE",
      "inputParameters": {
        "llmProvider": "openai",
        "model": "gpt-4o-mini",
        "messages": [
          {"role": "system", "message": "You are a professional report writer. Generate well-structured markdown reports."},
          {"role": "user", "message": "Write a detailed report about: ${workflow.input.topic}\nTarget audience: ${workflow.input.audience}"}
        ],
        "temperature": 0.7,
        "maxTokens": 2000
      }
    },
    {
      "name": "convert_to_pdf",
      "taskReferenceName": "pdf_output",
      "type": "GENERATE_PDF",
      "inputParameters": {
        "markdown": "${llm_report.output.result}",
        "pageSize": "A4",
        "theme": "default",
        "baseFontSize": 11,
        "pdfMetadata": {
          "title": "${workflow.input.topic}",
          "author": "Conductor AI Pipeline"
        }
      }
    }
  ],
  "outputParameters": {
    "reportMarkdown": "${llm_report.output.result}",
    "pdfLocation": "${pdf_output.output.result.location}"
  }
}
```

**注册并运行：**

```shell
curl -X POST 'http://localhost:8080/api/metadata/workflow' \
  -H 'Content-Type: application/json' \
  -d @llm_to_pdf_pipeline.json

curl -X POST 'http://localhost:8080/api/workflow/llm_to_pdf_pipeline' \
  -H 'Content-Type: application/json' \
  -d '{"topic": "Microservices observability best practices", "audience": "Platform engineering team"}'
```

---

### 网络搜索——实时信息检索

启用 LLM 内置的网络搜索，回答时事相关问题或查找最新信息。无需 MCP 服务器或外部工具——由提供商原生处理搜索。

```json
{
  "name": "web_search_workflow",
  "version": 1,
  "schemaVersion": 2,
  "inputParameters": ["question"],
  "tasks": [
    {
      "name": "web_search_chat",
      "taskReferenceName": "chat",
      "type": "LLM_CHAT_COMPLETE",
      "inputParameters": {
        "llmProvider": "openai",
        "model": "gpt-4o-mini",
        "messages": [
          {"role": "system", "message": "Use web search to find current information."},
          {"role": "user", "message": "${workflow.input.question}"}
        ],
        "webSearch": true,
        "maxTokens": 1000
      }
    }
  ],
  "outputParameters": {
    "answer": "${chat.output.result}"
  }
}
```

**注册并运行：**

```shell
curl -X POST 'http://localhost:8080/api/metadata/workflow' \
  -H 'Content-Type: application/json' \
  -d @web_search_workflow.json

curl -X POST 'http://localhost:8080/api/workflow/web_search_workflow' \
  -H 'Content-Type: application/json' \
  -d '{"question": "What are the latest developments in AI regulation?"}'
```

!!! note "提供商支持"
    OpenAI、Anthropic 和 Google Gemini 支持网络搜索。设置 `"webSearch": true`——同一参数在所有提供商中均适用。

---

### 代码执行——沙箱化代码解释器

让 LLM 在沙箱环境中编写并运行代码。适用于数据分析、计算、图表生成以及受益于可执行代码的任务。

```json
{
  "name": "code_execution_workflow",
  "version": 1,
  "schemaVersion": 2,
  "inputParameters": ["task"],
  "tasks": [
    {
      "name": "code_chat",
      "taskReferenceName": "chat",
      "type": "LLM_CHAT_COMPLETE",
      "inputParameters": {
        "llmProvider": "google_gemini",
        "model": "gemini-2.5-flash",
        "messages": [
          {"role": "system", "message": "Use code execution to compute results and analyze data."},
          {"role": "user", "message": "${workflow.input.task}"}
        ],
        "codeInterpreter": true,
        "maxTokens": 2000
      }
    }
  ],
  "outputParameters": {
    "result": "${chat.output.result}"
  }
}
```

**注册并运行：**

```shell
curl -X POST 'http://localhost:8080/api/metadata/workflow' \
  -H 'Content-Type: application/json' \
  -d @code_execution_workflow.json

curl -X POST 'http://localhost:8080/api/workflow/code_execution_workflow' \
  -H 'Content-Type: application/json' \
  -d '{"task": "Calculate the first 100 prime numbers and find the average gap between consecutive primes"}'
```

!!! note "提供商支持"
    OpenAI（`code_interpreter`）、Anthropic（`code_execution`）和 Google Gemini（`codeExecution`）支持代码执行。设置 `"codeInterpreter": true`——同一参数在所有提供商中均适用。

---

### 编码智能体——规划、编码与评审

一个三步智能体：规划实现方案，使用代码解释器编写并执行代码，然后评审结果。该模式适用于自动化代码生成任务。

```json
{
  "name": "coding_agent",
  "version": 1,
  "schemaVersion": 2,
  "inputParameters": ["task"],
  "tasks": [
    {
      "name": "plan",
      "taskReferenceName": "plan",
      "type": "LLM_CHAT_COMPLETE",
      "inputParameters": {
        "llmProvider": "openai",
        "model": "gpt-4o",
        "messages": [
          {"role": "system", "message": "Break down the coding task into clear numbered steps."},
          {"role": "user", "message": "${workflow.input.task}"}
        ],
        "temperature": 0.2,
        "maxTokens": 1000
      }
    },
    {
      "name": "write_and_run",
      "taskReferenceName": "code",
      "type": "LLM_CHAT_COMPLETE",
      "inputParameters": {
        "llmProvider": "openai",
        "model": "gpt-4o",
        "messages": [
          {"role": "system", "message": "Write the code, run it, verify the output, and fix any errors."},
          {"role": "user", "message": "Plan:\n${plan.output.result}\n\nTask: ${workflow.input.task}"}
        ],
        "codeInterpreter": true,
        "temperature": 0.1,
        "maxTokens": 4000
      }
    },
    {
      "name": "review",
      "taskReferenceName": "review",
      "type": "LLM_CHAT_COMPLETE",
      "inputParameters": {
        "llmProvider": "openai",
        "model": "gpt-4o-mini",
        "messages": [
          {"role": "system", "message": "Review the implementation for correctness and code quality."},
          {"role": "user", "message": "Task: ${workflow.input.task}\n\nCode:\n${code.output.result}"}
        ],
        "maxTokens": 1000
      }
    }
  ],
  "outputParameters": {
    "code": "${code.output.result}",
    "review": "${review.output.result}"
  }
}
```

**注册并运行：**

```shell
curl -X POST 'http://localhost:8080/api/metadata/workflow' \
  -H 'Content-Type: application/json' \
  -d @coding_agent.json

curl -X POST 'http://localhost:8080/api/workflow/coding_agent' \
  -H 'Content-Type: application/json' \
  -d '{"task": "Write a Python function that converts Roman numerals to integers, with unit tests"}'
```

---

### 扩展思考——复杂推理

在 LLM 生成最终回复前，为其提供用于逐步推理的 token 预算。适用于数学、逻辑、代码评审和复杂分析。

```json
{
  "name": "extended_thinking_workflow",
  "version": 1,
  "schemaVersion": 2,
  "inputParameters": ["problem"],
  "tasks": [
    {
      "name": "think_deeply",
      "taskReferenceName": "think",
      "type": "LLM_CHAT_COMPLETE",
      "inputParameters": {
        "llmProvider": "anthropic",
        "model": "claude-sonnet-4-20250514",
        "messages": [
          {"role": "user", "message": "${workflow.input.problem}"}
        ],
        "thinkingTokenLimit": 10000,
        "maxTokens": 16000
      }
    }
  ],
  "outputParameters": {
    "answer": "${think.output.result}"
  }
}
```

**注册并运行：**

```shell
curl -X POST 'http://localhost:8080/api/metadata/workflow' \
  -H 'Content-Type: application/json' \
  -d @extended_thinking_workflow.json

curl -X POST 'http://localhost:8080/api/workflow/extended_thinking_workflow' \
  -H 'Content-Type: application/json' \
  -d '{"problem": "Prove that the square root of 2 is irrational."}'
```

!!! note "提供商支持"
    Anthropic（`thinkingTokenLimit`）和 Google Gemini（`thinkingBudgetTokens`）支持扩展思考。OpenAI 使用 `"reasoningEffort": "high"` 获得类似效果。

---

### 使用 previousResponseId 的多轮对话链接

将多次 LLM 调用链接为一段对话，而无需重新发送完整的消息历史。第一次调用返回一个 `responseId`；将其作为 `previousResponseId` 传给下一次调用。OpenAI 的 Responses API 在服务器端存储对话，从而节省 token 和延迟。

```json
{
  "name": "multi_turn_chain",
  "description": "Two-step conversation using previousResponseId to avoid resending history",
  "version": 1,
  "schemaVersion": 2,
  "inputParameters": ["topic"],
  "tasks": [
    {
      "name": "first_turn",
      "taskReferenceName": "turn1",
      "type": "LLM_CHAT_COMPLETE",
      "inputParameters": {
        "llmProvider": "openai",
        "model": "gpt-4o",
        "messages": [
          {"role": "system", "message": "You are a technical architect. Be concise."},
          {"role": "user", "message": "Design a high-level architecture for: ${workflow.input.topic}"}
        ],
        "temperature": 0.3,
        "maxTokens": 2000
      }
    },
    {
      "name": "follow_up",
      "taskReferenceName": "turn2",
      "type": "LLM_CHAT_COMPLETE",
      "inputParameters": {
        "llmProvider": "openai",
        "model": "gpt-4o",
        "messages": [
          {"role": "user", "message": "Now list the key risks and mitigations for this architecture."}
        ],
        "previousResponseId": "${turn1.output.responseId}",
        "temperature": 0.3,
        "maxTokens": 2000
      }
    }
  ],
  "outputParameters": {
    "architecture": "${turn1.output.result}",
    "risks": "${turn2.output.result}"
  }
}
```

**注册并运行：**

```shell
curl -X POST 'http://localhost:8080/api/metadata/workflow' \
  -H 'Content-Type: application/json' \
  -d @multi_turn_chain.json

curl -X POST 'http://localhost:8080/api/workflow/multi_turn_chain' \
  -H 'Content-Type: application/json' \
  -d '{"topic": "Real-time collaborative document editor"}'
```

第二次调用只发送新的用户消息——OpenAI 已通过 `previousResponseId` 拥有完整的对话上下文。这在长智能体循环中特别有用，因为每次迭代都重新发送完整历史代价高昂。

!!! note "提供商支持"
    `previousResponseId` 受 OpenAI 和 Azure OpenAI（Responses API）支持。其他提供商需要在每次调用时发送完整的消息历史。

---

### 网络研究智能体——搜索、综合与 PDF

一个多步智能体：使用网络搜索收集信息，用带扩展思考的 LLM 综合生成报告，并将其转换为 PDF。在单个工作流中组合了三种内置能力。

```json
{
  "name": "web_research_agent",
  "version": 1,
  "schemaVersion": 2,
  "inputParameters": ["topic"],
  "tasks": [
    {
      "name": "gather_information",
      "taskReferenceName": "research",
      "type": "LLM_CHAT_COMPLETE",
      "inputParameters": {
        "llmProvider": "openai",
        "model": "gpt-4o",
        "messages": [
          {"role": "system", "message": "Use web search to find comprehensive, current information. Search for multiple perspectives and recent developments."},
          {"role": "user", "message": "Research this topic thoroughly: ${workflow.input.topic}"}
        ],
        "webSearch": true,
        "temperature": 0.3,
        "maxTokens": 3000
      }
    },
    {
      "name": "synthesize_report",
      "taskReferenceName": "report",
      "type": "LLM_CHAT_COMPLETE",
      "inputParameters": {
        "llmProvider": "anthropic",
        "model": "claude-sonnet-4-20250514",
        "messages": [
          {"role": "system", "message": "Synthesize the research into a well-structured markdown report with sections, key findings, and citations."},
          {"role": "user", "message": "Topic: ${workflow.input.topic}\n\nResearch:\n${research.output.result}\n\nWrite a comprehensive report."}
        ],
        "thinkingTokenLimit": 5000,
        "maxTokens": 8000
      }
    },
    {
      "name": "convert_to_pdf",
      "taskReferenceName": "pdf",
      "type": "GENERATE_PDF",
      "inputParameters": {
        "markdown": "${report.output.result}",
        "pageSize": "A4",
        "pdfMetadata": {
          "title": "${workflow.input.topic}",
          "author": "Conductor Research Agent"
        }
      }
    }
  ],
  "outputParameters": {
    "report": "${report.output.result}",
    "pdf": "${pdf.output.result.location}"
  }
}
```

**注册并运行：**

```shell
curl -X POST 'http://localhost:8080/api/metadata/workflow' \
  -H 'Content-Type: application/json' \
  -d @web_research_agent.json

curl -X POST 'http://localhost:8080/api/workflow/web_research_agent' \
  -H 'Content-Type: application/json' \
  -d '{"topic": "The state of WebAssembly adoption in 2026"}'
```

---

### AI 提供商配置 { #ai-provider-configuration }

在启动服务器之前设置环境变量。当对应的 API 密钥存在时，Conductor 会自动启用提供商。

```bash
# OpenAI (required for most examples)
export OPENAI_API_KEY=sk-your-openai-api-key

# Anthropic (for RAG, extended thinking examples)
export ANTHROPIC_API_KEY=sk-ant-your-anthropic-key

# Google Gemini — API key (simplest)
export GEMINI_API_KEY=your-gemini-api-key
# Or Vertex AI (for enterprise/GCP) — set project and location in application.properties
```

如需配置向量数据库和其他高级选项，添加到 `application.properties`：

```properties
# PostgreSQL Vector DB (for RAG examples)
conductor.vectordb.instances[0].name=postgres-prod
conductor.vectordb.instances[0].type=postgres
conductor.vectordb.instances[0].postgres.datasourceURL=jdbc:postgresql://localhost:5432/vectors
conductor.vectordb.instances[0].postgres.user=conductor
conductor.vectordb.instances[0].postgres.password=secret
conductor.vectordb.instances[0].postgres.dimensions=1536
```

---

## 更多示例

要查看更多 AI 工作流定义，请参阅 [GitHub 上的 AI 工作流示例](https://github.com/conductor-oss/conductor/tree/main/ai/examples)。
