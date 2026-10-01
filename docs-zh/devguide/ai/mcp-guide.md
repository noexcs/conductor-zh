---
description: "MCP (Model Context Protocol) 与 Conductor 的集成 — 把 AI agent 连接到外部工具、在运行时发现工具、带持久化重试地执行，以及把工作流暴露为 MCP 工具。"
---

# MCP 集成

<section class="integration-hero integration-hero--mcp" aria-label="MCP integration">
  <p><strong>Model Context Protocol (MCP)</strong> 是 agent 用来发现和调用工具的开放标准。在 Conductor 中，MCP 调用作为工作流任务运行：<code>LIST_MCP_TOOLS</code> 询问服务器提供了什么，<code>CALL_MCP_TOOL</code> 调用其中一个工具。由于每次调用都是一个任务，它获得与所有其他步骤相同的重试、可观测性和历史记录。</p>
  <div class="integration-action-grid integration-action-grid--three">
    <a class="integration-action-card" href="#list_mcp_tools-discover-available-tools">
      <span class="integration-action-card__title">发现工具</span>
      <span>在运行时检视一个 MCP 服务器的可用能力。</span>
    </a>
    <a class="integration-action-card" href="#call_mcp_tool-execute-a-tool">
      <span class="integration-action-card__title">调用工具</span>
      <span>把一个选定的工具作为原生 Conductor 任务运行。</span>
    </a>
    <a class="integration-action-card" href="#exposing-workflows-as-mcp-tools">
      <span class="integration-action-card__title">暴露工作流</span>
      <span>把持久化的工作流逻辑发布为 MCP 工具。</span>
    </a>
  </div>
</section>


## 什么是 MCP

MCP 定义了 AI agent 发现和使用工具的协议。不必硬编码 API 集成，你的 agent 向 MCP 服务器询问"你有哪些工具？"，并得到一个结构化的列表。Agent（或 LLM）挑选合适的工具，MCP 服务器执行它。

**没有 MCP：** 每个工具集成都是自定义代码 — 不同的认证、不同的 schema、不同的错误处理。

**有了 MCP：** 工具被标准化。连接一次，即可使用任何兼容 MCP 的工具服务器。

Conductor 将 MCP 作为一等集成来支持，提供两个原生系统任务。


## 原生 MCP 系统任务

### LIST_MCP_TOOLS — 发现可用工具

查询一个 MCP 服务器，返回它提供的工具列表，包括名称、描述和参数 schema。

```json
{
  "name": "discover_tools",
  "taskReferenceName": "discover",
  "type": "LIST_MCP_TOOLS",
  "inputParameters": {
    "mcpServer": "${workflow.input.mcpServerUrl}"
  }
}
```

**输出：** 带 schema 的结构化工具列表。可以把它直接传给 LLM，让它决定调用哪个工具。

**为什么重要：** 工具发现发生在运行时。你的 agent 在设计时无需知道存在哪些工具 — 它动态地发现它们。向 MCP 服务器添加一个新工具，所有使用它的 agent 立即获得该能力。


### CALL_MCP_TOOL — 执行一个工具

在 MCP 服务器上以给定参数调用一个特定工具。

```json
{
  "name": "execute_tool",
  "taskReferenceName": "execute",
  "type": "CALL_MCP_TOOL",
  "inputParameters": {
    "mcpServer": "${workflow.input.mcpServerUrl}",
    "method": "${plan.output.result.method}",
    "arguments": "${plan.output.result.arguments}"
  }
}
```

**Conductor 在裸 MCP 之上增加了什么：**

- **持久化执行** — 如果工具调用失败，Conductor 按任务的重试策略重试。重试是自动的且可配置的（固定延迟、指数退避、线性退避）。
- **完整审计轨迹** — 每次工具调用都被持久化：方法、参数、响应、耗时和重试历史。你可以准确检视你的 agent 做了什么。
- **崩溃恢复** — 如果服务器在两次工具调用之间崩溃，工作流从最后完成的步骤恢复。工具调用绝不会被悄悄丢失。
- **超时处理** — 配置 `responseTimeoutSeconds`，防止卡住的工具调用阻塞你的 agent。


## 连接 MCP 服务器

Conductor 通过 HTTP 连接任何 MCP 服务器。把服务器 URL 作为工作流输入传入，或硬编码在任务定义中。

```json
{
  "mcpServer": "http://localhost:3001/mcp"
}
```

### 使用多个 MCP 服务器

一个 agent 可以在同一个工作流中连接多个 MCP 服务器。从每个服务器发现工具，合并工具列表，让 LLM 在所有工具中选择：

```json
{
  "name": "multi_tool_agent",
  "version": 1,
  "schemaVersion": 2,
  "tasks": [
    {
      "name": "discover_github_tools",
      "taskReferenceName": "github_tools",
      "type": "LIST_MCP_TOOLS",
      "inputParameters": {
        "mcpServer": "http://localhost:3001/mcp"
      }
    },
    {
      "name": "discover_db_tools",
      "taskReferenceName": "db_tools",
      "type": "LIST_MCP_TOOLS",
      "inputParameters": {
        "mcpServer": "http://localhost:3002/mcp"
      }
    },
    {
      "name": "plan_with_all_tools",
      "taskReferenceName": "plan",
      "type": "LLM_CHAT_COMPLETE",
      "inputParameters": {
        "llmProvider": "openai",
        "model": "gpt-4o-mini",
        "messages": [
          {
            "role": "system",
            "message": "Available tools: GitHub: ${github_tools.output.tools}, Database: ${db_tools.output.tools}. User task: ${workflow.input.task}. Pick the best tool. Respond with JSON: {\"server\": \"github\" or \"db\", \"method\": \"tool_name\", \"arguments\": {}}"
          }
        ],
        "temperature": 0.1
      }
    }
  ]
}
```


## 将工作流暴露为 MCP 工具

任何 Conductor 工作流都可以通过 MCP Gateway 暴露为 MCP 工具。这意味着其他 agent 和 LLM 可以使用 MCP 协议发现并调用你的工作流。

```
Agent → LIST_MCP_TOOLS → discovers your workflow
Agent → CALL_MCP_TOOL → starts your workflow
Conductor → executes with full durability
Agent → receives structured output
```

你的工作流的 `inputParameters` 成为工具的输入 schema，`outputParameters` 成为工具的输出。工作流带着完整的持久化执行保证运行 — 重试、持久化、补偿 — 而在调用方 agent 看来只是一次简单的工具调用。

这形成了可组合的架构：工作流调用 MCP 工具，而工作流*本身就是* MCP 工具。Agent 可以调用其他 agent 的工作流，而无需知道它们是工作流。


## MCP 与 HTTP 与自定义工作者

| 方式 | 何时使用 |
|----------|-------------|
| **MCP** (`LIST_MCP_TOOLS` + `CALL_MCP_TOOL`) | 通过 MCP 服务器暴露的工具。动态工具发现。Agent 在运行时决定调用哪个工具。 |
| **HTTP** (`HTTP` 系统任务) | 端点已知的直接 API 调用。无需工具发现。 |
| **自定义工作者** (`SIMPLE` 任务) | 需要自定义代码的复杂业务逻辑。多步骤处理。 |

当你的 agent 需要**动态发现工具**，或你想在多个 agent 之间**标准化工具访问**时，MCP 是最佳选择。简单的、端点已知的 API 调用使用 HTTP。不适合塞进单次 API 调用的逻辑使用自定义工作者。


## 完整示例：带审批的 MCP agent

一个生产就绪的 agent：发现工具、规划、获取人工审批、执行并总结：

```json
{
  "name": "mcp_agent_with_approval",
  "description": "Discover tools, plan, execute with approval, summarize",
  "version": 1,
  "schemaVersion": 2,
  "inputParameters": ["task", "mcpServerUrl"],
  "tasks": [
    {
      "name": "list_available_tools",
      "taskReferenceName": "discover_tools",
      "type": "LIST_MCP_TOOLS",
      "inputParameters": {
        "mcpServer": "${workflow.input.mcpServerUrl}"
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
          {
            "role": "system",
            "message": "You are an AI agent. Available tools: ${discover_tools.output.tools}. User wants to: ${workflow.input.task}"
          },
          {
            "role": "user",
            "message": "Which tool should I use and what parameters? Respond with JSON: {\"method\": \"string\", \"arguments\": {}}"
          }
        ],
        "temperature": 0.1,
        "maxTokens": 500
      }
    },
    {
      "name": "human_review",
      "taskReferenceName": "approval",
      "type": "HUMAN",
      "inputParameters": {
        "plannedAction": "${plan.output.result}"
      }
    },
    {
      "name": "execute_tool",
      "taskReferenceName": "execute",
      "type": "CALL_MCP_TOOL",
      "inputParameters": {
        "mcpServer": "${workflow.input.mcpServerUrl}",
        "method": "${plan.output.result.method}",
        "arguments": "${plan.output.result.arguments}"
      }
    },
    {
      "name": "summarize_result",
      "taskReferenceName": "summarize",
      "type": "LLM_CHAT_COMPLETE",
      "inputParameters": {
        "llmProvider": "anthropic",
        "model": "claude-sonnet-4-20250514",
        "messages": [
          {
            "role": "user",
            "message": "The user asked: ${workflow.input.task}\n\nTool result: ${execute.output.content}\n\nSummarize this result for the user."
          }
        ],
        "maxTokens": 500
      }
    }
  ],
  "outputParameters": {
    "plan": "${plan.output.result}",
    "toolResult": "${execute.output.content}",
    "summary": "${summarize.output.result}",
    "approvedBy": "${approval.output.reviewer}"
  }
}
```

这里的每一种任务类型 — `LIST_MCP_TOOLS`、`LLM_CHAT_COMPLETE`、`CALL_MCP_TOOL`、`HUMAN` — 都是原生 Conductor 系统任务。不需要自定义代码。


## 后续步骤

- **[生产级 Agent 架构](production-agent-architecture.md)** — 在 agent 产生第一个结果之后治理和运营这个使用工具的 agent。
- **[构建你的第一个 Agent 工作流图](first-ai-agent.md)** — 把 SDK 编写的 agent 与持久化工作流任务组合起来。
- **[动态工作流](dynamic-workflows.md)** — 自行生成执行计划的 agent。
- **[Human-in-the-Loop](human-in-the-loop.md)** — MCP 工具调用的审批模式。
- **[LLM 编排](llm-orchestration.md)** — 12 个原生 LLM 提供方、向量数据库、内容生成。
