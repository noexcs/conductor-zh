---
description: "Conductor Agents——把 SDK 编写的 agent 编译为持久化、可检查的 Conductor 图，并作为可复用的 AGENT 任务使用。"
---

# Conductor Agents

<section class="integration-hero integration-hero--agents" aria-labelledby="conductor-agents-hero-title">
  <h2 id="conductor-agents-hero-title">上手 Conductor Agents</h2>
  <div class="integration-action-grid integration-action-grid--three">
    <a class="integration-action-card" href="../../quickstart/first-agent.html">
      <span class="integration-action-card__title">编写一个 Conductor Agent</span>
      <span>使用 Python、Java、TypeScript/JavaScript 或 C# 编写。</span>
    </a>
    <a class="integration-action-card" href="agent-framework-recipes.html">
      <span class="integration-action-card__title">引入一个框架 agent</span>
      <span>运行用 OpenAI Agents、Google ADK、LangChain、LangGraph 等构建的 agent。</span>
    </a>
    <a class="integration-action-card" href="#use-a-deployed-agent-in-a-workflow">
      <span class="integration-action-card__title">在工作流中使用</span>
      <span>把已部署的图作为可复用的 <code>AGENT</code> 任务调用。</span>
    </a>
  </div>
</section>

**Conductor Agent** 是你用代码编写并注册到服务端的 agent。你用 Conductor SDK 构建它，或从受支持的 agent 框架引入它，Conductor 把它编译为普通的工作流定义。由于编译后的 agent 就是工作流，每次 LLM 调用、工具调用、等待、重试和分支都在 UI 和 API 中可见，agent 也能与工作流可包含的其他一切组合：其他任务、分支、调度、人工审批和取消。Conductor Agents 支持 Python、Java、TypeScript/JavaScript 和 C#。

Conductor Agents 是构建 AI 行为的两种方式之一。另一种是[声明式 AI 工作流](llm-orchestration.md)——把 LLM、MCP 和控制流任务直接放进工作流定义。当你要构建的就是编排本身时，选择声明式路径；当 agent 逻辑在代码中、你想让它在一个持久化进程内运行时，选择 Conductor Agent。

## 服务端要求

在部署或调用 Conductor Agent 之前，在服务端启用 AI 集成：

```properties
conductor.integrations.ai.enabled=true
```

当此属性为 false 或缺省时，已部署 agent 的控制平面和 `agentType: "conductor"` 执行模式不可用。

## 生命周期

每个 Conductor Agent 都经历同样五个操作，下面的名称就是你在代码中会看到的 SDK 动词：

1. **创建（Create）**：在代码中定义 agent，来自 SDK 自己的 `Agent` 类或受支持的框架对象。
2. **计划（Plan）**：检查 agent 将编译成的工作流图。在开发和 CI 中、任何部署之前都很有用。
3. **部署（Deploy）**：把编译好的 agent 作为可复用、带版本的 Conductor Agent 注册到服务端。
4. **Serve**：启动执行 agent 工具的工作者进程（如果框架需要）。
5. **运行（Run）**：执行 agent。开发期间，`run` 一步完成编译和运行。生产环境中，工作流通过 `AGENT` 任务按名称调用已部署的 agent。

简言之：迭代时用 `run`，然后 `deploy` 和 `serve`，让工作流和其他调用方可以启动稳定的已部署版本。

框架特有的代码、包版本和可运行示例参见[框架 Agent](agent-framework-recipes.md)。服务端配置和凭证，请完成[连接到 Conductor](../../quickstart/connect.md)。

## 在工作流中使用已部署 agent { #use-a-deployed-agent-in-a-workflow }

`agentType` 选择的是**执行模式**，不是编写框架：

- `agentType: "a2a"`（默认）调用远程 A2A 端点。
- `agentType: "conductor"` 按 `name` 选择并运行一个已部署的 Conductor Agent。

OpenAI Agents、Google ADK、LangGraph 和其他受支持的框架是 SDK 编写路径。它们不是 `agentType` 的取值。

```json
{
  "name": "run_agent",
  "taskReferenceName": "run_agent_ref",
  "type": "AGENT",
  "inputParameters": {
    "agentType": "conductor",
    "name": "planner",
    "prompt": "${workflow.input.prompt}",
    "pollIntervalSeconds": 5
  }
}
```

全新调用时，`name` 和 `prompt` 必填。`version` 可选，用于固定已部署 agent 的版本；省略则使用最新版本。`sessionId`、`runId`、`context`、`media`、`model`、`timeoutSeconds` 和 `idempotencyKey` 在已部署 agent 的契约需要时可用。未提供幂等键时，运行时会创建一个重启稳定的幂等键。

## 输出与持久化执行契约

`AGENT` 任务会写入 `executionId`、`agentName`、`state`、`text`，以及完成运行时的结构化 `output`。其 `state` 是归一化的 A2A 生命周期值：`working`、`input-required`、`completed`、`failed` 或 `canceled`。

| 运行时状态 / 输出 `state` | Conductor 任务状态 | 含义 |
|---|---|---|
| `RUNNING` / `working` | `IN_PROGRESS` | 任务在 `pollIntervalSeconds`（默认 5）后再轮询一次。 |
| `WAITING` / `input-required` | `COMPLETED` | 运行暂停等待人工或工具输入；输出包含 `waiting: true`，可能包含 `pendingTool`。 |
| `COMPLETED` / `completed` | `COMPLETED` | 输出包含最终 `text` 和结构化 `output`。 |
| `FAILED` / `failed` | `FAILED` | 任务包含完成原因。 |
| `CANCELED` / `canceled` | `CANCELED` | 任务在可用时包含取消原因。 |

`maxDurationSeconds` 约束完整运行（默认 86400 秒），`maxPollFailures` 约束连续瞬时轮询失败次数（默认 30）。两者都会让任务以终止性失败结束，并尽力取消子执行。这些保护独立于常规任务定义的超时。

## 恢复与取消

当 agent 等待外部输入时，它第一个 `AGENT` 任务会完成，而不是持有工作者。工作流可以用 `HUMAN` 任务收集答案，再用另一个 `AGENT` 任务恢复同一次运行：

```json
{
  "name": "resume_agent",
  "taskReferenceName": "resume_agent_ref",
  "type": "AGENT",
  "inputParameters": {
    "agentType": "conductor",
    "executionId": "${run_agent_ref.output.executionId}",
    "prompt": "${collect_answer_ref.output.answer}"
  }
}
```

恢复时，`executionId` 标识运行，`prompt` 提供回复；`name` 不是必需的。工作流取消会尽力传播给进行中的 Conductor Agent。

## 护栏与评估

SDK 编写的 agent 可以编译出针对 agent 输出以及工具输入或输出的运行时护栏。格式、PII 和已知危险模式选确定性的 regex 护栏；语义策略用 LLM 护栏；策略需要应用服务时用自定义或外部护栏。护栏可以重试、失败关闭（fail closed）、提供自定义修复，或暂停进行持久化的人工评审。把最强的护栏直接放在重要工具调用之前。

提升之前，评估录制到的 agent 行为——而不只是其最终文本。Python SDK 的评估框架可以断言工具选择和参数、交接、护栏事件、轮次计数和终止状态，然后用可选的 LLM 评判器评估定性标准。运行时策略和 CI 模式参见 [Agent 护栏](agent-guardrails.md) 和 [Agent 评估](agent-evals.md)。

## 工作流集成配方

仓库中的这些示例有意只包含稳定的工作流契约。它们与框架无关；用你所在框架的 Conductor SDK 创建并部署 `planner` / `researcher`。

| 配方 | 演示内容 |
|---|---|
| [`31-conductor-agent-basic.json`](https://github.com/conductor-oss/conductor/blob/main/ai/examples/31-conductor-agent-basic.json) | 可复用已部署 agent 作为工作流中的一个步骤。 |
| [`32-conductor-agent-human-in-loop.json`](https://github.com/conductor-oss/conductor/blob/main/ai/examples/32-conductor-agent-human-in-loop.json) | `WAITING` → `HUMAN` → 用 `executionId` 恢复。 |
| [`33-conductor-agent-multi-agent.json`](https://github.com/conductor-oss/conductor/blob/main/ai/examples/33-conductor-agent-multi-agent.json) | `FORK_JOIN` / `JOIN` 图内的并行专家 agent。 |
| [`34-conductor-agent-cancel.json`](https://github.com/conductor-oss/conductor/blob/main/ai/examples/34-conductor-agent-cancel.json) | 来自父图的取消传播。 |

接下来：在[框架 Agent](agent-framework-recipes.md)中选择一个框架路线，在[构建你的第一个 Agentic 工作流图](first-ai-agent.md)中组合已部署 agent，然后用[生产 Agent 架构](production-agent-architecture.md)处理治理、评估、部署、恢复和运维。
