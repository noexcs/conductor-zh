---
description: "为什么 AI agent 选 Conductor — 原生 LLM 与 MCP 任务、持久化人工审批、受治理的运行时路径，以及运维恢复。"
---

# 为什么 agent 选 Conductor

agent 在生产环境中会因为普通的原因失败。进程在循环中途崩溃，一次工具调用失败就丢掉整次运行，事后也没人能看到是哪个决策导致了哪个动作。Conductor 通过把 agent 的每个步骤 — 每次模型调用、每次工具调用 — 都作为持久化工作流任务运行来解决这个问题。失败的步骤会被重试，被中断的运行从最后完成的步骤恢复，运行的完整历史被记录下来。

本页展示使用原生任务时这在实践中的样子。当你改用框架编写的 agent 时，同样的性质同样适用；参见 [Conductor Agents](conductor-agents.md)。


## 把 LLM 调用作为工作流任务

一次 LLM 调用就是一个系统任务。提供方、模型和消息都是普通的任务输入：

```json
{
  "name": "plan_action",
  "type": "LLM_CHAT_COMPLETE",
  "inputParameters": {
    "llmProvider": "anthropic",
    "model": "claude-sonnet-4-20250514",
    "messages": [
      {"role": "system", "message": "You are a planning agent. Tools: ${tools.output}"},
      {"role": "user", "message": "${workflow.input.goal}"}
    ],
    "temperature": 0.1,
    "maxTokens": 1000
  }
}
```

Conductor 把任务输入、结果、提供方返回的 token 用量和任务结局与工作流执行一起记录下来。为每个任务选择提供方和模型；受维护的能力矩阵见 [LLM 编排](llm-orchestration.md)。


## 发现并调用工具 — 原生 MCP

MCP (Model Context Protocol) 是 agent 工具使用的开放标准。在 Conductor 上，工具发现和执行都是系统任务：

```json
[
  {
    "name": "discover",
    "type": "LIST_MCP_TOOLS",
    "inputParameters": {
      "mcpServer": "http://localhost:3001/mcp"
    }
  },
  {
    "name": "execute",
    "type": "CALL_MCP_TOOL",
    "inputParameters": {
      "mcpServer": "http://localhost:3001/mcp",
      "method": "${plan.output.result.method}",
      "arguments": "${plan.output.result.arguments}"
    }
  }
]
```

agent 在运行时发现工具，LLM 挑选一个已批准的方法，Conductor 记录调用、任务结局和结果。把 MCP 与[护栏](agent-guardrails.md)组合，可以约束能力选择，并强制要求有重大影响的动作先经审批。


## Human-in-the-loop — 一行代码，永久持久

agent 在高风险动作之前需要人工审批。在 Conductor 上：

```json
{
  "name": "approval_gate",
  "type": "HUMAN",
  "inputParameters": {
    "action": "${plan.output.result.action}",
    "reasoning": "${plan.output.result.reasoning}"
  }
}
```

工作流暂停，直到审批完成或被拒绝。审批负载成为持久化的任务输出，执行在等待期间可以被检视和管理。


## Agent 循环 — 每次迭代做检查点

自主 agent 会循环：规划、行动、观察、重复。在 Conductor 上，每次迭代都是一个持久化检查点：

```json
{
  "name": "agent_loop",
  "taskReferenceName": "loop",
  "type": "DO_WHILE",
  "loopCondition": "if ($.think['result']['route'] == 'done' || $.loop['iteration'] >= 20) { false; } else { true; }",
  "loopOver": [
    {
      "name": "think",
      "type": "LLM_CHAT_COMPLETE",
      "inputParameters": {
        "llmProvider": "anthropic",
        "model": "claude-sonnet-4-20250514",
        "messages": [
          {"role": "system", "message": "Goal: ${workflow.input.goal}. Respond with JSON only: {\"route\": \"call_tool\", \"action\": \"tool_name\", \"arguments\": {}} or {\"route\": \"done\", \"answer\": \"final answer\"}."}
        ],
        "jsonOutput": true
      }
    },
    {
      "name": "act_or_finish",
      "taskReferenceName": "act_or_finish",
      "type": "SWITCH",
      "evaluatorType": "value-param",
      "expression": "route",
      "inputParameters": {
        "route": "${think.output.result.route}"
      },
      "decisionCases": {
        "call_tool": [
          {
            "name": "act",
            "taskReferenceName": "act",
            "type": "CALL_MCP_TOOL",
            "inputParameters": {
              "mcpServer": "${workflow.input.mcpServerUrl}",
              "method": "${think.output.result.action}",
              "arguments": "${think.output.result.arguments}"
            }
          }
        ],
        "done": []
      },
      "defaultCase": []
    }
  ]
}
```

如果某个后续任务失败，已完成任务的输出仍然保留在执行记录中。循环条件强制迭代上限；任务投递是至少一次，因此工具自身仍需保证幂等。


## 动态工作流 — LLM 生成执行计划

LLM 或某个服务可以生成一个完整的工作流定义 JSON，并把它作为运行时计划提交：

```json
{
  "name": "execute_agent_plan",
  "type": "START_WORKFLOW",
  "inputParameters": {
    "startWorkflow": {
          "workflowDef": "${planner_llm.output.result}",
      "input": "${workflow.input.taskInput}"
    }
  }
}
```

LLM 的输出是数据，而不是对运行中执行的不受限变更。启动之前先校验该定义及其被允许的能力。生成的工作流使用与已注册定义相同的持久化状态、重试策略和执行控制。

与 `DYNAMIC` 任务（在运行时解析一个已批准的任务）和 `FORK_JOIN_DYNAMIC`（在运行时创建经过校验、有界的并行分支）结合，Conductor 让运行时计划作为数据可检视、可治理。

当运行时计划需要自己的执行边界、审计轨迹、版本和生命周期时，使用这个模式。


## RAG 管道 — 原生向量数据库支持

检索增强生成就是两个系统任务，无需外部框架：

```json
[
  {
    "name": "search",
    "type": "LLM_SEARCH_INDEX",
    "inputParameters": {
      "vectorDB": "postgres-prod",
      "namespace": "kb",
      "index": "articles",
      "embeddingModelProvider": "openai",
      "embeddingModel": "text-embedding-3-small",
      "query": "${workflow.input.question}"
    }
  },
  {
    "name": "answer",
    "type": "LLM_CHAT_COMPLETE",
    "inputParameters": {
      "llmProvider": "anthropic",
      "model": "claude-sonnet-4-20250514",
      "messages": [
        {"role": "system", "message": "Answer based on: ${search.output.result}"},
        {"role": "user", "message": "${workflow.input.question}"}
      ]
    }
  }
]
```

Pinecone、pgvector 和 MongoDB Atlas 通过向量工作流任务获得支持。当检索只是图的一部分时，同样的模式可以与现有的 agent 框架组合。


## 多 agent 委派 — 带生命周期的子工作流

父 agent 把任务委派给专家 agent。每个专家都是一个带完整生命周期管理的子工作流：

```json
{
  "name": "parallel_research",
  "type": "FORK_JOIN_DYNAMIC",
  "inputParameters": {
    "dynamicTasks": "${planner.output.result.research_tasks}",
    "dynamicTasksInput": "${planner.output.result.task_inputs}"
  },
  "dynamicForkTasksParam": "dynamicTasks",
  "dynamicForkTasksInputParamName": "dynamicTasksInput"
}
```

LLM 决定生成多少个研究 agent、每个调查什么。Conductor 在运行时创建分支、并行运行它们，并汇合结果。如果一个分支失败，它独立重试而不影响其他分支。父 agent 能看到完整的执行树 — 在 UI 中从父到子、再到子子孙逐级下钻。


## 长时运行的工作流 — 演进而不断裂

一个 agent 工作流可以运行数天。让定义变更保持显式和版本化，这样随着系统演进，执行行为仍然是可理解的。

```json
{
  "name": "agent_workflow",
  "version": 2,
  "tasks": [
    {"name": "plan", "type": "LLM_CHAT_COMPLETE", "...": "..."},
    {"name": "validate", "type": "INLINE", "...": "..."},
    {"name": "execute", "type": "CALL_MCP_TOOL", "...": "..."}
  ]
}
```

运行中的执行保留它们开始时的定义版本；新的执行可以被导向新版本。如果新定义必须应用到已经开始的工作上，[重启该执行](../../architecture/durable-execution.md#replay-and-recovery)要刻意进行，并评估其副作用。


## 失败是图的显式组成部分

Conductor 记录任务状态，并暴露重试、超时、失败工作流、暂停、恢复和终止控制。把失败策略构建进图中，而不是把它当作事后补救。

保证如下：

- **至少一次任务投递** — 每个任务在执行之前都持久化到持久化存储。如果 worker 崩溃，任务被自动重新入队并投递给另一个 worker。任务不会消失。
- **Sweeper 恢复** — 一个后台 sweeper 服务持续扫描停滞的任务。如果一个任务处于 `IN_PROGRESS` 但其 worker 已失去响应（没有心跳、超过 `responseTimeoutSeconds`），sweeper 会把它重新入队。如果 Conductor 服务器本身重启，sweeper 在启动时恢复所有在途的工作。
- **可配置的重试策略** — 每个任务都有重试次数、延迟和退避策略。重试由引擎管理，而不是你的代码。指数退避、固定延迟和线性退避都是内置的。
- **失败工作流** — 当工作流用尽重试后失败时，`failureWorkflow` 自动运行。补偿逻辑放在这里：撤销 API 调用、释放资源、发送告警。失败工作流拥有关于什么失败了、为什么失败的完整上下文。
- **终止处理** — 使用终止状态、工作流超时和告警，让结果对运维人员可操作。

```json
{
  "name": "critical_agent",
  "failureWorkflow": "agent_failure_handler",
  "tasks": [
    {
      "name": "risky_action",
      "type": "CALL_MCP_TOOL",
      "retryCount": 5,
      "retryLogic": "EXPONENTIAL_BACKOFF",
      "retryDelaySeconds": 10,
      "responseTimeoutSeconds": 30,
      "timeoutPolicy": "RETRY"
    }
  ]
}
```

配置重试和补偿时，要考虑每个外部系统的幂等行为。工作流会记录结局和失败路径，供运维人员检视。


## 显式编排，普通工作者

JSON 定义让图结构、任务输入和工作流策略可见。把业务逻辑和副作用放进内置任务或工作者，然后根据外部系统的幂等契约设计重试和补偿。这种分离让执行路径更容易检视、版本化，并作为经过校验的数据来生成。


## 可观测性 — 自动的，而非可选

每个 `LLM_CHAT_COMPLETE` 任务自动记录：

- 完整的 prompt（对话中的每一条消息）
- 完整的响应
- Token 用量（prompt token、补全 token、总计）
- 模型和提供方
- 延迟
- 重试历史（如有）

每个 `CALL_MCP_TOOL` 任务记录方法、参数、响应和耗时。每个 `HUMAN` 任务记录谁批准了、何时、带什么负载。所有这些都可以通过 API 查询，并在 UI 中可见。

使用执行视图和 API，把这些任务级记录与图路径和重试历史一起检视。


## Agent 用例矩阵

每一种 agent 模式都映射到一个具体的 Conductor 原语：

| 用例 | Conductor 模式 |
|---|---|
| **工具调用 agent** | `LLM_CHAT_COMPLETE` + `CALL_MCP_TOOL` |
| **审批把关的动作** | `HUMAN` 任务 + `SWITCH` 处理超时 |
| **规划者/执行者循环** | `DO_WHILE` + `SET_VARIABLE` |
| **多 agent 委派** | `SUB_WORKFLOW` 或 `FORK_JOIN_DYNAMIC` |
| **长等待外部系统** | `HUMAN` 或 `WAIT` 任务 |
| **高扇出研究** | `FORK_JOIN_DYNAMIC` + `JOIN` |
| **RAG 管道** | `LLM_SEARCH_INDEX` + `LLM_CHAT_COMPLETE` |
| **内容生成** | `GENERATE_IMAGE` / `GENERATE_AUDIO` / `GENERATE_VIDEO` / `GENERATE_PDF` |
| **自建计划的 agent** | `LLM_CHAT_COMPLETE` + 内联定义的 `START_WORKFLOW` |
| **确定性后处理** | `INLINE`（JavaScript）或 `JSON_JQ_TRANSFORM` |


## 后续步骤

- **[Conductor Agents](conductor-agents.md)** — 编写 Conductor Agent，或把现有框架 agent 带入持久化的 Conductor 图。
- **[框架 Agents](agent-framework-recipes.md)** — OpenAI Agents、Google ADK、LangChain、LangGraph、Vercel AI SDK 和 Conductor Agents 的受支持 SDK 路径。
- **[生产级 Agent 架构](production-agent-architecture.md)** — 标准的端到端 agent 模式，完整接线。
- **[AI Agent 的失败语义](failure-semantics.md)** — 每一种场景下的精确失败契约。
- **[构建你的第一个 Agent 工作流图](first-ai-agent.md)** — 把 SDK 编写的 agent 与持久化工作流任务组合起来。
- **[Token 效率](token-efficiency.md)** — 持久化执行如何节省 token、降低 LLM 成本。
