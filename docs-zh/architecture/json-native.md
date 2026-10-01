---
description: Conductor 以 JSON 形式存储工作流定义——这是该持久化执行工作流引擎的标准运行时格式。在运行时创建动态工作流，对定义进行版本管理和差异对比，并将任意工作流暴露为 API 或 MCP 工具。
---

# JSON + 代码原生的工作流编排

Conductor 以 JSON 形式存储工作流定义。这不是 UI 上的便利功能，也不是简化模式。JSON 就是标准的运行时表示。每一个工作流，无论通过 SDK、API、UI 还是文件创建，都以 JSON 文档的形式被存储、版本化并执行。

## “JSON + 代码原生”在机制上意味着什么

你可以直接用 JSON 编写[工作流定义](../documentation/configuration/workflowdef/index.md)，也可以借助 [SDK](../documentation/clientsdks/index.md) 用代码编写。两者的产物相同：一份 JSON 文档。当你用代码定义工作流时，SDK 会将其转换为该 JSON 并向服务器注册。服务器始终只存储、版本化和执行 JSON。无论你以哪种方式编写工作流，下文内容都适用。

1. **存储。** 工作流定义是一份[持久化在数据存储中](durable-execution.md#what-persists)的 JSON 文档。执行引擎读取该文档来调度任务。
2. **版本控制。** 每个[版本](../devguide/how-tos/Workflows/versioning-workflows.md)都是一份独立的 JSON 文档。多个版本可以并发运行。运行中的执行使用启动时保存的快照，对后续变更保持不可变。
3. **API 一致性。** 你在文件中编写的 JSON，与发送到 [API](../documentation/api/metadata.md) 的、在 UI 中看到的、以及从 SDK 返回的 JSON 完全一致。不存在编译后的中间形式。
4. **动态创建。** 你可以[在运行时以 JSON 对象构造工作流定义](../devguide/cookbook/dynamic-workflows.md)，并直接传给 [`StartWorkflowRequest` API](../documentation/api/startworkflow.md)。Conductor 无需预先注册即可立即执行。

## 动态工作流详解

Conductor 支持三个层次的运行时灵活性。

### 1. 动态工作流定义

在 `StartWorkflowRequest` 中传入完整的工作流定义：

```json
{
  "name": "dynamic_agent_plan",
  "workflowDef": {
    "name": "dynamic_agent_plan",
    "tasks": [
      {
        "name": "search_web",
        "taskReferenceName": "search",
        "type": "HTTP",
        "inputParameters": {
          "http_request": {
            "uri": "https://api.search.com/query",
            "method": "POST",
            "body": { "q": "${workflow.input.query}" }
          }
        }
      },
      {
        "name": "summarize",
        "taskReferenceName": "summarize",
        "type": "SIMPLE"
      }
    ]
  },
  "input": {
    "query": "conductor workflow engine"
  }
}
```

无需预先注册。定义内嵌在执行中并被持久化。

### 2. 动态任务

[`DYNAMIC`](../documentation/configuration/workflowdef/operators/dynamic-task.md) 任务类型在运行时解析要执行哪个任务：

```json
{
  "name": "run_tool",
  "taskReferenceName": "tool_call",
  "type": "DYNAMIC",
  "inputParameters": {
    "taskToExecute": "${plan.output.nextTool}"
  },
  "dynamicTaskNameParam": "taskToExecute"
}
```

`taskToExecute` 的值来自前一个任务的输出，例如由 LLM 选择某个工具。Conductor 会在运行时解析并调度该任务。

### 3. 动态 fork/join

[`FORK_JOIN_DYNAMIC`](../documentation/configuration/workflowdef/operators/dynamic-fork-task.md) 操作符在运行时创建并行分支：

```json
{
  "name": "parallel_tool_calls",
  "taskReferenceName": "fork",
  "type": "FORK_JOIN_DYNAMIC",
  "inputParameters": {
    "dynamicTasks": "${plan.output.parallelTasks}",
    "dynamicTasksInput": "${plan.output.taskInputs}"
  },
  "dynamicForkTasksParam": "dynamicTasks",
  "dynamicForkTasksInputParamName": "dynamicTasksInput"
}
```

分支数量、各分支的任务类型及其输入都在运行时决定。fork 之后需要接一个 [`JOIN`](../documentation/configuration/workflowdef/operators/join-task.md)。如果分支列表由智能体生成，在执行计划之前要对其进行校验并强制分支数量上限。

[子工作流](../documentation/configuration/workflowdef/operators/sub-workflow-task.md)同样可以在运行时被选择和参数化。对于运行时生成计划的受治理实现——包括能力白名单、有界扇出和审批——参见 [Durable Adaptive Graphs](../devguide/ai/dynamic-workflows.md)。

## 结构上即确定性

JSON 定义描述运行什么、按什么顺序运行。它不包含可执行代码，因此自身无法打开数据库连接、写文件或调用 API。所有副作用都发生在[工作者](../devguide/concepts/workers.md)或[系统任务](../documentation/configuration/workflowdef/systemtasks/index.md)内部，在那里它们是隔离的、可测试的、可独立部署的。定义本身只是惰性数据。

正因为定义是惰性的，执行才是确定性的。给定相同的输入，Conductor 每次都以相同的顺序调度相同的任务。没有环境状态，也没有隐藏的变更。这就是[回放](durable-execution.md#replay-and-recovery)能无条件成立的原因：重启几个月前启动的工作流，它会重新执行相同的图。把编排嵌入应用代码的引擎，只能通过限制你的代码被允许做什么来承诺这一点。

同样的划分也让编排与实现彼此独立。顺序、分支、重试和超时存在于定义中；实现逻辑存在于工作者中，可以是任何语言。你可以更改工作者而不触及工作流，也可以更改工作流而无需重新部署工作者。

## 为什么这对智能体很重要

### 智能体产出结构化输出，而 JSON 是其原生形式

LLM 已经以函数调用和 JSON 响应的形式产出结构化输出。Conductor 的工作流定义是同一种对象。因此 LLM 可以直接生成工作流定义。你的应用校验计划并施加其[策略边界](../devguide/ai/agent-guardrails.md)，然后 Conductor 执行它。

### 无需编译/部署的运行时生成

大多数引擎要求先修改代码、编译并部署，之后新工作流才能运行。Conductor 不需要。规划智能体以 JSON 生成定义，你的代码将其连同内嵌的定义一起发送到 [`POST /api/workflow`](../documentation/api/startworkflow.md)，Conductor 立即校验、持久化并执行。其结果与任何预先注册的工作流一样持久、可观测、可重试。

### 可检查性与可审计性

每次执行都会记录其使用的定义快照、每个任务的输入、输出、状态和重试历史，以及工作流自身的输入、输出和状态转换。你可以查询、对比、导出并[回放](durable-execution.md#replay-and-recovery)任何一次执行。对于智能体工作流，这份记录展示了智能体规划了什么、调用了哪些工具、模型返回了什么，以及[人批准了什么](../devguide/ai/human-in-the-loop.md)。

### 支持差异对比的版本控制

由于定义是 JSON，它们理应纳入版本控制。你可以在拉取请求（pull request）中审查变更，对比两个版本以查看具体改了什么，并通过[重新注册较早的版本来回滚](../devguide/how-tos/Workflows/versioning-workflows.md)。多个版本可以并行运行，这让灰度发布（canary rollout）变得直接可行。运行中的执行永远不受其中任何影响，因为每个执行都保留启动时保存的快照。

## 将工作流暴露为 API 和 MCP 工具

任何 Conductor 工作流本身就是一个 API 端点：

```bash
# Start a workflow (async, returns execution ID)
conductor workflow start -w my_agent -i '{"query": "summarize this document"}'

# Get the result
conductor workflow status {executionId}
```

??? note "使用 cURL"
    ```bash
    curl -X POST http://localhost:8080/api/workflow/my_agent \
      -H 'Content-Type: application/json' \
      -d '{"query": "summarize this document"}'

    curl http://localhost:8080/api/workflow/{executionId}
    ```

工作流会返回其 `outputParameters` 声明的结构化输出，因此服务和智能体可以像调用任何其他 API 一样调用它。工作流还可以[注册为 MCP 工具](../devguide/ai/mcp-guide.md)，让 LLM 和智能体框架以结构化的输入和输出来发现并调用它。

## 后续步骤

- **[持久化执行语义](durable-execution.md)** &mdash; 哪些内容被持久化、哪些会被重试、故障矩阵。
- **[智能体与 AI](../devguide/ai/index.md)** &mdash; 什么是智能体，以及它们如何在 Conductor 上运行。
- **[从 JSON 运行工作流](../quickstart/first-workflow.md)** &mdash; 使用 CLI 注册并运行 JSON 工作流。
- **[工作流定义参考](../documentation/configuration/workflowdef/index.md)** &mdash; 工作流定义的完整 JSON schema。
- **[动态 Fork](../documentation/configuration/workflowdef/operators/dynamic-fork-task.md)** &mdash; 运行时决定的并行执行。
