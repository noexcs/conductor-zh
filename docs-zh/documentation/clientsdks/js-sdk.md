---
description: "使用工作流管理和任务轮询，用 JavaScript/TypeScript 构建 Conductor 工作者。"
source_repo: "https://github.com/conductor-oss/javascript-sdk"
sdk_page: javascript
---

# JavaScript SDK

## 安装 SDK

```shell
npm install @io-orkes/conductor-javascript
```

## 60 秒快速开始

**步骤 1：创建工作流**

工作流是引用任务类型的定义。我们将构建一个名为 `greetings` 的工作流，它运行一个工作者任务并返回其输出。

```typescript
import { ConductorWorkflow, simpleTask } from "@io-orkes/conductor-javascript";

const workflow = new ConductorWorkflow(executor, "greetings")
  .add(simpleTask("greet_ref", "greet", { name: "${workflow.input.name}" }))
  .outputParameters({ result: "${greet_ref.output.result}" });

await workflow.register();
```

**步骤 2：编写一个工作者**

工作者是使用 `@worker` 装饰的 TypeScript 函数，它们轮询 Conductor 以获取任务并执行。

```typescript
import { worker } from "@io-orkes/conductor-javascript";

@worker({ taskDefName: "greet" })
async function greet(task: Task) {
  return {
    status: "COMPLETED",
    outputData: { result: `Hello ${task.inputData.name}` },
  };
}
```

**步骤 3：运行你的第一个工作流应用**

创建如下内容的 `quickstart.ts`：

```typescript
import {
  OrkesClients,
  ConductorWorkflow,
  TaskHandler,
  worker,
  simpleTask,
} from "@io-orkes/conductor-javascript";
import type { Task } from "@io-orkes/conductor-javascript";

// A worker is any TypeScript function.
@worker({ taskDefName: "greet" })
async function greet(task: Task) {
  return {
    status: "COMPLETED" as const,
    outputData: { result: `Hello ${task.inputData.name}` },
  };
}

async function main() {
  // Configure the SDK (reads CONDUCTOR_SERVER_URL / CONDUCTOR_AUTH_* from env).
  const clients = await OrkesClients.from();
  const executor = clients.getWorkflowClient();

  // Build a workflow with the fluent builder.
  const workflow = new ConductorWorkflow(executor, "greetings")
    .add(simpleTask("greet_ref", "greet", { name: "${workflow.input.name}" }))
    .outputParameters({ result: "${greet_ref.output.result}" });

  await workflow.register();

  // Start polling for tasks (auto-discovers @worker decorated functions).
  const handler = new TaskHandler({
    client: clients.getClient(),
    scanForDecorated: true,
  });
  await handler.startWorkers();

  // Run the workflow and get the result.
  const run = await workflow.execute({ name: "Conductor" });
  console.log(`result: ${run.output?.result}`);

  await handler.stopWorkers();
}

main();
```

运行它：

```shell
npx ts-node quickstart.ts
```

就这样——你定义了一个工作者、构建了一个工作流并执行了它。打开你所配置的 Conductor 服务器的 UI，即可检查该执行（实例）。

## 你可以构建什么 { #what-you-can-build }

SDK 为常见的编排模式提供了类型化的构建器。以下是你可以串联起来的能力的预览：

**从工作流发起 HTTP 调用**——无需编写工作者即可调用任意 API（[kitchensink.ts](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/kitchensink.ts)）：

```typescript
httpTask("call_api", {
  uri: "https://api.example.com/orders/${workflow.input.orderId}",
  method: "POST",
  body: { items: "${workflow.input.items}" },
  headers: { "Authorization": "Bearer ${workflow.input.token}" },
})
```

**任务之间的等待**——将工作流暂停一段时间或暂停到某个时间戳（[kitchensink.ts](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/kitchensink.ts)）：

```typescript
.add(simpleTask("step1_ref", "process_order", {...}))
.add(waitTaskDuration("cool_down", "10s"))        // wait 10 seconds
.add(simpleTask("step2_ref", "send_confirmation", {...}))
```

**并行执行（fork/join）**——扇出到多个分支再会合（[fork-join.ts](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/advanced/fork-join.ts)）：

```typescript
workflow.fork([
  [simpleTask("email_ref", "send_email", {})],
  [simpleTask("sms_ref", "send_sms", {})],
  [simpleTask("push_ref", "send_push", {})],
])
```

**条件分支**——根据输入值路由（[kitchensink.ts](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/kitchensink.ts)）：

```typescript
switchTask("route_ref", "${workflow.input.tier}", {
  premium: [simpleTask("fast_ref", "fast_track", {})],
  standard: [simpleTask("normal_ref", "standard_process", {})],
})
```

**子工作流**——用更小的工作流组合出工作流（[sub-workflows.ts](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/advanced/sub-workflows.ts)）：

```typescript
const child = new ConductorWorkflow(executor, "payment_flow").add(...);
const parent = new ConductorWorkflow(executor, "order_flow")
  .add(child.toSubWorkflowTask("pay_ref"));
```

以上所有功能都是类型安全的、可组合的，并作为 JSON 注册到服务器——工作者可以使用任何语言编写。

## 工作者

工作者是执行 Conductor 任务的 TypeScript 函数。用 `@worker` 装饰任意函数即可将其注册为工作者（由 `TaskHandler` 自动发现），并将其用作工作流任务。

```typescript
import { worker, TaskHandler } from "@io-orkes/conductor-javascript";

@worker({ taskDefName: "greet", concurrency: 5, pollInterval: 100 })
async function greet(task: Task) {
  return {
    status: "COMPLETED",
    outputData: { result: `Hello ${task.inputData.name}` },
  };
}

@worker({ taskDefName: "process_payment", domain: "payments" })
async function processPayment(task: Task) {
  const result = await paymentGateway.charge(task.inputData.customerId, task.inputData.amount);
  return { status: "COMPLETED", outputData: { transactionId: result.id } };
}

// Auto-discover and start all decorated workers
const handler = new TaskHandler({ client, scanForDecorated: true });
await handler.startWorkers();

// Graceful shutdown
process.on("SIGTERM", async () => {
  await handler.stopWorkers();
  process.exit(0);
});
```

**工作者配置：**

```typescript
@worker({
  taskDefName: "my_task",    // Required: task name
  concurrency: 5,             // Max concurrent tasks (default: 1)
  pollInterval: 100,          // Polling interval in ms (default: 100)
  domain: "production",       // Task domain for multi-tenancy
  workerId: "worker-123",     // Unique worker identifier
})
```

**环境变量覆盖**（无需修改代码）：

```shell
# Global (all workers)
export CONDUCTOR_WORKER_ALL_POLL_INTERVAL=500
export CONDUCTOR_WORKER_ALL_CONCURRENCY=10

# Per-worker override
export CONDUCTOR_WORKER_SEND_EMAIL_CONCURRENCY=20
export CONDUCTOR_WORKER_PROCESS_PAYMENT_DOMAIN=payments
```

**NonRetryableException**——将失败标记为终止性错误以防止重试：

```typescript
import { NonRetryableException } from "@io-orkes/conductor-javascript";

@worker({ taskDefName: "validate_order" })
async function validateOrder(task: Task) {
  const order = await getOrder(task.inputData.orderId);
  if (!order) {
    throw new NonRetryableException("Order not found"); // FAILED_WITH_TERMINAL_ERROR
  }
  return { status: "COMPLETED", outputData: { validated: true } };
}
```

- `throw new Error()` → 任务状态：`FAILED`（会重试）
- `throw new NonRetryableException()` → 任务状态：`FAILED_WITH_TERMINAL_ERROR`（不重试）

**使用 TaskContext 的长时间运行任务**——返回 `IN_PROGRESS` 可在外部进程完成期间保持任务存活。Conductor 会在指定间隔后回调（[task-context.ts](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/task-context.ts)）：

```typescript
import { worker, getTaskContext } from "@io-orkes/conductor-javascript";

@worker({ taskDefName: "process_video" })
async function processVideo(task: Task) {
  const ctx = getTaskContext();
  ctx?.addLog("Starting video processing...");

  if (!isComplete(task.inputData)) {
    ctx?.setCallbackAfter(30); // check again in 30 seconds
    return { status: "IN_PROGRESS", callbackAfterSeconds: 30 };
  }

  return { status: "COMPLETED", outputData: { url: "..." } };
}
```

`TaskContext` 也可用于一次性工作者——使用 `ctx?.addLog()` 流式输出可在 Conductor UI 中查看的日志。

**事件监听器**用于可观测性：

```typescript
const handler = new TaskHandler({
  client,
  scanForDecorated: true,
  eventListeners: [{
    onTaskExecutionCompleted(event) {
      metrics.histogram("task_duration_ms", event.durationMs, { task_type: event.taskType });
    },
    onTaskUpdateFailure(event) {
      alertOps({ severity: "CRITICAL", message: `Task update failed`, taskId: event.taskId });
    },
  }],
});
```

通过模块导入**跨文件组织工作者**：

```typescript
const handler = await TaskHandler.create({
  client,
  importModules: ["./workers/orderWorkers", "./workers/paymentWorkers"],
});
await handler.startWorkers();
```

**旧版 TaskManager API** 继续可用，并保持完全的向后兼容性。新项目应使用上文提到的 `@worker` + `TaskHandler`。

## 监控工作者

使用内置的 `MetricsCollector` 启用 Prometheus 指标：

```typescript
import { MetricsCollector, MetricsServer, TaskHandler } from "@io-orkes/conductor-javascript";

const metrics = new MetricsCollector();
const server = new MetricsServer(metrics, 9090);
await server.start();

const handler = new TaskHandler({
  client,
  eventListeners: [metrics],
  scanForDecorated: true,
});
await handler.startWorkers();
// GET http://localhost:9090/metrics — Prometheus text format
// GET http://localhost:9090/health  — {"status":"UP"}
```

收集 18 种指标：轮询次数、执行时长、错误率、输出大小等——并提供 p50/p75/p90/p95/p99 分位数。完整参考见 [METRICS.md](https://github.com/conductor-oss/javascript-sdk/blob/main/METRICS.md)。

## 管理工作流执行

工作流注册完成后（见[你可以构建什么](#what-you-can-build)），你可以通过完整的生命周期运行和管理它：

```typescript
const executor = clients.getWorkflowClient();

// Start (async — returns immediately)
const workflowId = await executor.startWorkflow({
  name: "order_flow",
  input: { orderId: "ORDER-123" },
});

// Execute (sync — waits for completion)
const result = await workflow.execute({ orderId: "123" });

// Lifecycle management
await executor.pause(workflowId);
await executor.resume(workflowId);
await executor.terminate(workflowId, "cancelled by user");
await executor.restart(workflowId);
await executor.retry(workflowId);

// Signal a running WAIT task
await executor.signal(workflowId, TaskResultStatusEnum.COMPLETED, { approved: true });

// Search workflows
const results = await executor.search("workflowType = 'order_flow' AND status = 'RUNNING'");
```

可运行的示例（覆盖所有生命周期操作）见 [workflow-ops.ts](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/workflow-ops.ts)。

## 故障排查

- **工作者停止轮询或崩溃：** 默认情况下 `TaskHandler` 会监控并重启工作者轮询循环。使用 `handler.running` 和 `handler.runningWorkerCount` 暴露健康检查。如果启用了指标，请对 `worker_restart_total` 设置告警。
- **HTTP/2 连接错误：** 在可用时，SDK 使用 Undici 处理 HTTP/2。如果你的环境中长连接不稳定，SDK 会自动回退到 HTTP/1.1。你也可以提供自定义的 fetch 函数：`orkesConductorClient(config, myFetch)`。
- **任务卡在 SCHEDULED：** 确保你的工作者正在轮询正确的 `taskDefName`。工作者必须先于工作流执行之前启动。

## 示例

完整目录见[示例指南](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/README.md)。主要示例：

| Example | Description | Run |
|---------|-------------|-----|
| [workers-e2e.ts](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/workers-e2e.ts) | 端到端：3 个串联工作者并带验证 | `npx ts-node examples/workers-e2e.ts` |
| [quickstart.ts](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/quickstart.ts) | 60 秒入门：@worker + 工作流 + 执行 | `npx ts-node examples/quickstart.ts` |
| [kitchensink.ts](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/kitchensink.ts) | 一个工作流中的所有主要任务类型 | `npx ts-node examples/kitchensink.ts` |
| [workflow-ops.ts](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/workflow-ops.ts) | 生命周期：暂停、恢复、终止、重试、搜索 | `npx ts-node examples/workflow-ops.ts` |
| [test-workflows.ts](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/test-workflows.ts) | 使用模拟输出的单元测试（无工作者） | `npx ts-node examples/test-workflows.ts` |
| [metrics.ts](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/metrics.ts) | Prometheus 指标 + 运行在 :9090 的 HTTP 服务器 | `npx ts-node examples/metrics.ts` |
| [express-worker-service.ts](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/express-worker-service.ts) | 在同一进程中使用 Express.js + 工作者 | `npx ts-node examples/express-worker-service.ts` |
| [function-calling.ts](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/agentic-workflows/function-calling.ts) | LLM 动态选择调用哪个工作者 | `npx ts-node examples/agentic-workflows/function-calling.ts` |
| [fork-join.ts](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/advanced/fork-join.ts) | 带 join 同步的并行分支 | `npx ts-node examples/advanced/fork-join.ts` |
| [sub-workflows.ts](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/advanced/sub-workflows.ts) | 使用子工作流进行工作流组合 | `npx ts-node examples/advanced/sub-workflows.ts` |
| [human-tasks.ts](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/advanced/human-tasks.ts) | 人机协同：认领、更新、完成 | `npx ts-node examples/advanced/human-tasks.ts` |

## API 旅程示例

覆盖每个领域全部 API 的端到端示例：

| Example | APIs | Run |
|---------|------|-----|
| [authorization.ts](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/api-journeys/authorization.ts) | 授权 API（17 次调用） | `npx ts-node examples/api-journeys/authorization.ts` |
| [metadata.ts](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/api-journeys/metadata.ts) | 元数据 API（21 次调用） | `npx ts-node examples/api-journeys/metadata.ts` |
| [prompts.ts](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/api-journeys/prompts.ts) | 提示（Prompt）API（9 次调用） | `npx ts-node examples/api-journeys/prompts.ts` |
| [schedules.ts](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/api-journeys/schedules.ts) | 调度 API（13 次调用） | `npx ts-node examples/api-journeys/schedules.ts` |
| [secrets.ts](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/api-journeys/secrets.ts) | 密钥 API（12 次调用） | `npx ts-node examples/api-journeys/secrets.ts` |
| [integrations.ts](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/api-journeys/integrations.ts) | 集成 API（22 次调用） | `npx ts-node examples/api-journeys/integrations.ts` |
| [schemas.ts](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/api-journeys/schemas.ts) | 模式（Schema）API（10 次调用） | `npx ts-node examples/api-journeys/schemas.ts` |
| [applications.ts](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/api-journeys/applications.ts) | 应用 API（20 次调用） | `npx ts-node examples/api-journeys/applications.ts` |
| [event-handlers.ts](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/api-journeys/event-handlers.ts) | 事件处理器 API（18 次调用） | `npx ts-node examples/api-journeys/event-handlers.ts` |

## AI 与 LLM 工作流

Conductor 支持 AI 原生工作流，包括智能体式工具调用、RAG 管道和多智能体编排。SDK 为所有 LLM 任务类型提供了类型化的构建器：

| Builder | Description |
|---------|-------------|
| `llmChatCompleteTask` | LLM 聊天补全（OpenAI、Anthropic 等） |
| `llmTextCompleteTask` | 文本补全 |
| `llmGenerateEmbeddingsTask` | 生成向量嵌入 |
| `llmIndexDocumentTask` | 将文档索引到向量存储 |
| `llmIndexTextTask` | 将文本索引到向量存储 |
| `llmSearchIndexTask` | 搜索向量索引 |
| `llmSearchEmbeddingsTask` | 按嵌入相似度搜索 |
| `llmStoreEmbeddingsTask` | 存储预先计算的嵌入 |
| `llmQueryEmbeddingsTask` | 查询嵌入 |
| `generateImageTask` | 生成图像 |
| `generateAudioTask` | 生成音频 |
| `callMcpToolTask` | 调用 MCP 工具 |
| `listMcpToolsTask` | 列出可用的 MCP 工具 |

**示例：LLM 聊天工作流**

```typescript
import { ConductorWorkflow, llmChatCompleteTask, Role } from "@io-orkes/conductor-javascript";

const workflow = new ConductorWorkflow(executor, "ai_chat")
  .add(llmChatCompleteTask("chat_ref", "openai", "gpt-4o", {
    messages: [{ role: Role.USER, message: "${workflow.input.question}" }],
    temperature: 0.7,
    maxTokens: 500,
  }))
  .outputParameters({ answer: "${chat_ref.output.result}" });

await workflow.register();
const run = await workflow.execute({ question: "What is Conductor?" });
console.log(run.output?.answer);
```

**智能体式工作流**

构建 AI 智能体，让 LLM 动态选择并调用 TypeScript 工作者作为工具。
所有示例见 [examples/agentic-workflows/](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/agentic-workflows/)。

| Example | Description |
|---------|-------------|
| [llm-chat.ts](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/agentic-workflows/llm-chat.ts) | 两个 LLM 之间的自动化多轮对话 |
| [llm-chat-human-in-loop.ts](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/agentic-workflows/llm-chat-human-in-loop.ts) | 通过 WAIT 任务等待人工输入的交互式聊天 |
| [function-calling.ts](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/agentic-workflows/function-calling.ts) | LLM 动态选择调用哪个工作者函数 |
| [mcp-weather-agent.ts](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/agentic-workflows/mcp-weather-agent.ts) | 用于实时数据的 MCP 工具发现与调用 |
| [multiagent-chat.ts](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/agentic-workflows/multiagent-chat.ts) | 多智能体辩论：乐观派对阵怀疑派，由主持人主持 |

**RAG 与向量数据库工作流**

| Example | Description |
|---------|-------------|
| [rag-workflow.ts](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/advanced/rag-workflow.ts) | 端到端 RAG：文档索引 → 语义搜索 → LLM 回答 |
| [vector-db.ts](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/advanced/vector-db.ts) | 向量数据库操作：嵌入生成、存储、搜索 |

## 文档

| Document | Description |
|----------|-------------|
| [SDK 开发指南](https://github.com/conductor-oss/javascript-sdk/blob/main/SDK_DEVELOPMENT.md) | 架构、模式、陷阱、测试 |
| [指标参考](https://github.com/conductor-oss/javascript-sdk/blob/main/METRICS.md) | 全部 18 个 Prometheus 指标及其说明 |
| [破坏性变更](https://github.com/conductor-oss/javascript-sdk/blob/main/BREAKING_CHANGES.md) | v3.x 迁移指南 |
| [工作流管理](https://github.com/conductor-oss/javascript-sdk/blob/main/docs/api-reference/workflow-executor.md) | 启动、暂停、恢复、终止、重试、搜索、发送信号 |
| [任务管理](https://github.com/conductor-oss/javascript-sdk/blob/main/docs/api-reference/task-client.md) | 任务操作、日志、队列管理 |
| [元数据](https://github.com/conductor-oss/javascript-sdk/blob/main/docs/api-reference/metadata-client.md) | 任务与工作流定义、标签、速率限制 |
| [调度](https://github.com/conductor-oss/javascript-sdk/blob/main/docs/api-reference/scheduler-client.md) | 使用 CRON 表达式进行工作流调度 |
| [应用](https://github.com/conductor-oss/javascript-sdk/blob/main/docs/api-reference/application-client.md) | 应用管理、访问密钥、角色 |
| [事件](https://github.com/conductor-oss/javascript-sdk/blob/main/docs/api-reference/event-client.md) | 事件处理器、事件驱动的工作流 |
| [人工任务](https://github.com/conductor-oss/javascript-sdk/blob/main/docs/api-reference/human-executor.md) | 人机协同工作流、表单模板 |
| [服务注册](https://github.com/conductor-oss/javascript-sdk/blob/main/docs/api-reference/service-registry-client.md) | 服务发现、熔断器 |

## 支持

- 针对 SDK 的 bug、问题和功能请求，请[提交 Issue（SDK）](https://github.com/conductor-oss/javascript-sdk/issues)
- 针对 Conductor OSS 服务器的问题，请[提交 Issue（Conductor server）](https://github.com/conductor-oss/conductor/issues)
- 参与社区讨论和寻求帮助，请[加入 Conductor Slack](https://join.slack.com/t/orkes-conductor/shared_invite/zt-2vdbx239s-Eacdyqya9giNLHfrCavfaA)
- 问答交流请见 [Orkes 社区论坛](https://community.orkes.io/)

## 许可证

Apache 2.0


## Examples

Browse all examples on GitHub: [conductor-oss/javascript-sdk/examples](https://github.com/conductor-oss/javascript-sdk/tree/main/examples)

| Example | Type |
|---|---|
| [Readme](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/README.md) | file |
| [Advanced](https://github.com/conductor-oss/javascript-sdk/tree/main/examples/advanced) | directory |
| [Agentic Workflows](https://github.com/conductor-oss/javascript-sdk/tree/main/examples/agentic-workflows) | directory |
| [Api Journeys](https://github.com/conductor-oss/javascript-sdk/tree/main/examples/api-journeys) | directory |
| [Dynamic Workflow](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/dynamic-workflow.ts) | file |
| [Event Listeners](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/event-listeners.ts) | file |
| [Express Worker Service](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/express-worker-service.ts) | file |
| [Helloworld](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/helloworld.ts) | file |
| [Kitchensink](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/kitchensink.ts) | file |
| [Metrics](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/metrics.ts) | file |
| [Perf Test](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/perf-test.ts) | file |
| [Quickstart](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/quickstart.ts) | file |
| [Task Configure](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/task-configure.ts) | file |
| [Task Context](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/task-context.ts) | file |
| [Test Workflows](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/test-workflows.ts) | file |
| [Worker Configuration](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/worker-configuration.ts) | file |
| [Workers E2E](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/workers-e2e.ts) | file |
| [Workflow Ops](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/workflow-ops.ts) | file |
