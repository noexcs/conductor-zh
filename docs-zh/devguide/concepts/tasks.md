---
description: "了解 Conductor 中的任务——工作流的可复用构建块，包括系统任务、工作者任务、运算符、支持 14+ AI 提供商的 LLM 任务，以及 MCP 工具调用。"
---

# 任务

<section class="concept-hero concept-hero--tasks">
  <svg class="concept-hero__graphic" viewBox="0 20 440 150" role="img" aria-label="A workflow routes work to a system task or worker task and receives a recorded output">
    <defs><marker id="task-arrow" markerWidth="8" markerHeight="8" refX="7" refY="4" orient="auto"><path d="M0,0 L8,4 L0,8 Z" fill="currentColor" /></marker></defs>
    <rect x="14" y="68" width="105" height="54" rx="10" class="concept-hero__node" />
    <text x="66" y="91" text-anchor="middle" class="concept-hero__label">工作流</text>
    <text x="66" y="108" text-anchor="middle" class="concept-hero__detail">输入</text>
    <path d="M119 95 H163 V54 H191" class="concept-hero__line" marker-end="url(#task-arrow)" />
    <path d="M163 95 V136 H191" class="concept-hero__line" marker-end="url(#task-arrow)" />
    <rect x="199" y="28" width="122" height="52" rx="10" class="concept-hero__node concept-hero__node--accent" />
    <text x="260" y="51" text-anchor="middle" class="concept-hero__label">系统任务</text>
    <text x="260" y="68" text-anchor="middle" class="concept-hero__detail">HTTP · WAIT · LLM</text>
    <rect x="199" y="110" width="122" height="52" rx="10" class="concept-hero__node" />
    <text x="260" y="133" text-anchor="middle" class="concept-hero__label">工作者任务</text>
    <text x="260" y="150" text-anchor="middle" class="concept-hero__detail">你的代码</text>
    <path d="M321 54 H350 V95 H360" class="concept-hero__line" marker-end="url(#task-arrow)" />
    <path d="M321 136 H350 V95" class="concept-hero__line" />
    <rect x="366" y="68" width="66" height="54" rx="10" class="concept-hero__outcome-box" />
    <text x="399" y="100" text-anchor="middle" class="concept-hero__label">输出</text>
  </svg>
</section>

一个**任务**是 Conductor 工作流的基本构建块。它们可复用且模块化，代表应用中的步骤，例如处理数据文件、调用 AI 模型或执行某些逻辑。

在 Conductor 中，任务可以被定义、配置然后执行。下面了解不同但相关的概念：**任务定义**、**任务配置**和**任务执行**。


## 任务类型

任务分为三类，让你能够使用预构建任务、自定义逻辑或两者组合来灵活地构建工作流：

### 系统任务

Conductor 自带 20 多个 [系统任务](../../documentation/configuration/workflowdef/systemtasks/index.md)——为常见用途设计的内置通用任务，例如调用 HTTP 端点、发布事件或运行 AI 推理。

系统任务由 Conductor 管理，并在其服务器的 JVM 内执行，让你无需编写自定义工作者即可上手。

| 类别 | 任务 |
|---|---|
| **核心** | HTTP, Inline (script), Event, Wait, Human, Kafka Publish, JSON JQ Transform, No Op |
| **流程控制** | Fork/Join, Dynamic Fork, Join, Switch, Do While, Sub Workflow, Start Workflow, Set Variable, Terminate, Dynamic |
| **AI / LLM** | Chat Completion, Text Completion, Embeddings, Vector Search, Content Generation, MCP Tool Calling |

## 常用系统任务

| 任务 | 类型 | 用途 |
|---|---|---|
| [HTTP](../../documentation/configuration/workflowdef/systemtasks/http-task.md) | `HTTP` | 调用 HTTP 或 REST 端点。 |
| [Event](../../documentation/configuration/workflowdef/systemtasks/event-task.md) | `EVENT` | 发布到事件目标或消息系统。 |
| Chat Completion | `LLM_CHAT_COMPLETE` | 对话式 AI 和可选的模型工具调用。 |
| [Wait](../../documentation/configuration/workflowdef/systemtasks/wait-task.md) | `WAIT` | 暂停直到某个时间、时长或外部信号。 |
| [JSON JQ Transform](../../documentation/configuration/workflowdef/systemtasks/json-jq-transform-task.md) | `JSON_JQ_TRANSFORM` | 重塑、过滤或聚合 JSON 数据。 |
| [Inline](../../documentation/configuration/workflowdef/systemtasks/inline-task.md) | `INLINE` | 用于验证或简单逻辑的小型服务器端 GraalJS 表达式。 |

完整的 [系统任务参考](../../documentation/configuration/workflowdef/systemtasks/index.md) 列出了每个内置任务及其配置。

### 工作者任务

工作者任务（`SIMPLE`）可用于实现 Conductor 系统任务范围之外的自定义逻辑。也称为 Simple 任务，工作者任务由你的任务工作者实现，它们运行在与 Conductor 分离的环境中。

一个最小工作者任务配置及其对应的 Python 工作者：

```json
{
  "name": "process_payment",
  "taskReferenceName": "process_payment_ref",
  "type": "SIMPLE",
  "inputParameters": {
    "orderId": "${workflow.input.orderId}",
    "amount": "${workflow.input.amount}"
  }
}
```

```python
@worker_task(task_definition_name="process_payment")
def process_payment(orderId: str, amount: float) -> dict:
    result = payment_gateway.charge(orderId, amount)
    return {"transactionId": result.id, "status": result.status}
```

### 运算符
[运算符](../../documentation/configuration/workflowdef/operators/index.md) 是内置的控制流原语，类似于循环、switch 分支或 fork/join 等编程语言结构。与系统任务一样，运算符也由 Conductor 管理。

| 运算符 | 用途 |
|---|---|
| [Do While](../../documentation/configuration/workflowdef/operators/do-while-task.md) | Do-while 循环 / For 循环 |
| [Dynamic](../../documentation/configuration/workflowdef/operators/dynamic-task.md) | 函数指针 |
| [Dynamic Fork](../../documentation/configuration/workflowdef/operators/dynamic-fork-task.md) | 动态并行执行 |
| [Fork](../../documentation/configuration/workflowdef/operators/fork-task.md) | 静态并行执行 |
| [Join](../../documentation/configuration/workflowdef/operators/join-task.md) | 映射 |
| [Set Variable](../../documentation/configuration/workflowdef/operators/set-variable-task.md) | 工作流变量声明 |
| [Start Workflow](../../documentation/configuration/workflowdef/operators/start-workflow-task.md) | 入口点 |
| [Sub Workflow](../../documentation/configuration/workflowdef/operators/sub-workflow-task.md) | 子程序 |
| [Switch](../../documentation/configuration/workflowdef/operators/switch-task.md) | Switch / If..then...else 选择 |
| [Terminate](../../documentation/configuration/workflowdef/operators/terminate-task.md) | 退出 |

完整的配置和示例参见 [运算符参考](../../documentation/configuration/workflowdef/operators/index.md)。


## 任务定义

[任务定义](../../documentation/configuration/taskdef.md) 用于定义任务的默认参数，例如输入和输出键、超时和重试。这提供了跨工作流的可复用性，因为在任务配置被写入工作流定义时，将引用已注册的任务定义。

```json
{
  "name": "process_payment",
  "retryCount": 3,
  "retryLogic": "EXPONENTIAL_BACKOFF",
  "retryDelaySeconds": 5,
  "maxRetryDelaySeconds": 60,
  "backoffJitterMs": 2000,
  "totalTimeoutSeconds": 300,
  "timeoutSeconds": 120,
  "responseTimeoutSeconds": 60,
  "pollTimeoutSeconds": 30
}
```

- **retryCount / retryLogic / retryDelaySeconds** — 失败任务重试多少次、回退策略，以及重试之间的初始延迟。
- **maxRetryDelaySeconds** — 为计算出的回退延迟设置上限。防止指数增长变得任意大。
- **backoffJitterMs** — 为每次重试延迟添加随机毫秒数，将并发重试在时间上分散开（防止惊群效应）。
- **totalTimeoutSeconds** — 跨越所有重试尝试的硬墙钟预算。一旦超过，无论 `retryCount` 如何，都不再尝试重试。
- **timeoutSeconds** — 任务被标记为 `TIMED_OUT` 之前，每次单独尝试允许的最大墙钟时间。
- **responseTimeoutSeconds** — 工作者领取任务后等待其响应的最长时间。适用于检测无响应的工作者。
- **pollTimeoutSeconds** — 工作者可以持有长轮询连接的最长时间，之后服务器会释放它。

使用工作者任务（`SIMPLE`）时，其任务定义必须先注册到 Conductor 服务器，然后才能在工作流中执行。由于系统任务由 Conductor 管理，除非你想自定义其默认参数，否则无需为系统任务添加任务定义。


## 任务配置 { #task-configuration }

任务配置存储在工作流定义的 `tasks` 数组中，构成描述工作流特定蓝图的组成部分：

- 任务的顺序和控制流。
- 数据如何通过任务输入和输出从一个任务传递到另一个任务。
- 其他工作流特定行为，例如可选性、缓存和 schema 强制。

每个任务的具体配置因任务类型而异。对于系统任务和运算符，任务配置将包含控制任务行为的重要参数。例如，HTTP 任务的任务配置将指定端点 URL 及其模板化的负载，这些将在任务执行时使用。

数据使用 `${...}` 表达式语法在任务之间传递。这允许任务引用之前任务的输出、工作流输入或其他上下文变量：

```json
{
  "name": "send_notification",
  "taskReferenceName": "send_notification_ref",
  "type": "SIMPLE",
  "inputParameters": {
    "recipient": "${workflow.input.email}",
    "paymentId": "${process_payment_ref.output.transactionId}",
    "status": "${process_payment_ref.output.status}"
  }
}
```

对于工作者任务（`SIMPLE`），配置将只包含其输入/输出以及对任务定义名称的引用，因为其行为的逻辑已经在你应用的工作者代码中指定。

每个工作流定义必须至少配置一个任务。

## 任务执行

任务执行对象在运行时创建，当时机是输入被传入已配置的任务时。该对象具有唯一 ID，代表任务操作的结果，包括任务状态、开始时间和输入/输出。


## AI 与 LLM 任务

Conductor 通过其 AI/LLM [系统任务](../../documentation/configuration/workflowdef/systemtasks/index.md) 提供对构建 AI 驱动工作流的一级支持。

### 支持的 LLM 提供商

Conductor 开箱即支持与 **14+ LLM 提供商** 集成：

Anthropic、OpenAI、Azure OpenAI、Google Gemini、AWS Bedrock、Mistral、Cohere、HuggingFace、Ollama、Perplexity、Grok、StabilityAI 等等。

每个提供商在服务器级别配置一次；工作流按名称引用它们，因此无需更改工作流逻辑即可轻松更换模型。

### MCP 工具调用

**LIST_MCP_TOOLS** 和 **CALL_MCP_TOOL** 系统任务让你的工作流可以发现并调用任何 MCP 兼容服务器暴露的工具。这使 LLM 智能体能够通过标准化协议与外部 API、数据库和服务交互。

### 向量数据库与 RAG

对于检索增强生成（RAG），Conductor 支持包括 **Pinecone**、**pgvector** 和 **MongoDB Atlas** 在内的向量存储。Embeddings 和 Vector Search 系统任务处理嵌入生成和相似度搜索步骤，使 RAG 流水线可以表示为标准工作流。

### 内容生成

除了文本，Conductor 的 AI 任务还支持生成图像、音频、视频和 PDF——适用于从 LLM 输出生成富媒体内容的工作流。

对于将 LLM 推理与工具使用相结合的全链路 AI 智能体模式，参见 [智能体文档](../ai/index.md)。
