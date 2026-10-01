---
description: "关于 Conductor 的常见问题：持久化工作流、自适应智能体、AI 编排、自托管、运维以及运行时控制。"
---

# 常见问题

## 通用

### Conductor 是开源的吗？

是的。Conductor 是一个完全开源的工作流引擎，以 Apache 2.0 许可证发布。你可以在自己的基础设施上自托管——没有供应商锁定，没有专有运行时，也不依赖任何云。自托管工作流引擎支持 5 种持久化后端、6 种消息代理，并可以在任何 Docker 或 JVM 能运行的地方运行。

### 这和 Netflix Conductor 是同一个东西吗？

是的。Conductor OSS 是 Netflix 将该项目捐赠给开源基金会之后，对原始 Netflix Conductor 仓库的延续。

### Netflix Conductor 被弃置了吗？

没有。原始的 Netflix 仓库已过渡到 Conductor OSS，这是该项目的新的家。积极的开发和维护在这里持续进行。

### 这个项目还在积极维护吗？

是的。Orkes 是该仓库的主要维护者，并在所有主流云提供商上提供 Conductor 的企业 SaaS 平台。

### Orkes Conductor 与 Conductor OSS 兼容吗？

100% 兼容。Orkes Conductor 构建于 Conductor OSS 之上，确保开源版本与企业版本之间的完全兼容。

### 工作流总是异步的吗？

不是。虽然 Conductor 擅长异步编排，但当需要即时结果时，它也支持同步工作流执行。

### 我需要使用 Conductor 专属的框架吗？

完全不需要。Conductor 与语言和框架无关。使用你喜欢的语言和框架——SDK 为 Java、Python、JavaScript、Go、C# 等提供了原生集成。

### Conductor 是低代码/无代码平台吗？

不是。Conductor 为写代码的开发者而设计。虽然工作流可以用 JSON 定义，但真正的威力来自于用你喜欢的编程语言构建工作者和任务。

### Conductor 能处理复杂工作流吗？

能。Conductor 支持高级模式，包括嵌套循环、动态分支、子工作流，以及包含数千个任务的工作流。

## Conductor 提供什么？

Conductor 将持久化工作流执行与内置系统任务、JSON 原生工作流定义、多语言工作者，以及原生 AI 和 MCP 能力结合在一起。用它来协调分布式服务、框架编写的智能体和自适应运行时路径，同时保留可检查的执行记录。

### JSON 不是太有限，无法表达复杂工作流吗？

不会。JSON 定义表达的是编排数据：任务顺序、输入、输出、操作符和策略。把副作用放进内置任务或工作者中，在那里它们可以被观测和重试。图保持机器可读且有版本记录，而工作者仍然是普通的代码。

对于运行时选择的路径，使用 [DYNAMIC 任务](../documentation/configuration/workflowdef/operators/dynamic-task.md)、[FORK_JOIN_DYNAMIC](../documentation/configuration/workflowdef/operators/dynamic-fork-task.md) 和 [子工作流](../documentation/configuration/workflowdef/operators/sub-workflow-task.md)。生成的定义是启动之前必须经过校验的数据；参见 [持久化自适应图](ai/dynamic-workflows.md)。

### 我能用 Conductor 做工作流自动化吗？

可以。Conductor 是一个开发者优先的工作流自动化平台——它不是低代码拖拽工具，而是一个代码优先的工作流引擎：你把工作流定义为代码或 JSON，并用任意语言实现任务工作者。它很适合自动化业务流程、数据管道，以及需要持久化执行和完整可观测性的多服务工作流。

## Conductor 能编排 AI 智能体吗？

能。Conductor 提供 LLM 任务、MCP 工具发现与调用、人工审批、向量工作流和自适应控制流。智能体可以在运行时选择已批准的路径，而 Conductor 保留执行的状态、任务结果，以及围绕执行的操作者控制。

## Conductor 支持 MCP（Model Context Protocol）吗？

支持。LIST_MCP_TOOLS 从任意 MCP 服务器发现可用工具，CALL_MCP_TOOL 执行它们。工作流也可以通过 MCP Gateway 暴露为 MCP 工具。

## Conductor 支持哪些 LLM 提供商？

参见 [LLM 编排](ai/llm-orchestration.md) 获取有来源依据的提供商矩阵和按能力划分的任务参考。提供商、模型和支持的特性各自独立演进，因此该矩阵是权威文档。

## Conductor 支持向量数据库和 RAG 吗？

支持。内置对 Pinecone、pgvector 和 MongoDB Atlas Vector Search 的支持。系统任务处理嵌入生成、存储、索引和语义搜索——使 RAG 流水线成为标准工作流。

## Conductor 是持久化执行引擎吗？

是的。Conductor 持久化工作流和任务状态，支持可配置的重试和超时策略，并为工作者和基础设施故障提供恢复路径。至少一次的任务投递意味着有副作用的工具必须是幂等的。参见 [持久化执行](../architecture/durable-execution.md)。

## Conductor 能处理百万级工作流吗？

能。Conductor 最初在 Netflix 构建以应对超大规模，它可以跨多个服务器实例水平扩展。工作者独立扩展，服务器在多种持久化后端上支持数百万个并发工作流执行。这种水平扩展架构使 Conductor 适合任意规模的生产工作流部署。

## Conductor 支持 Saga 模式吗？

支持。配置一个 `failureWorkflow`，在主工作流失败时运行补偿逻辑。结合任务级重试和超时策略，Conductor 为分布式事务提供完整的 Saga 模式支持。参见 [错误处理](how-tos/Workflows/handling-errors.md)。

## 我能创建运行时的工作流吗

可以。工作流定义是 JSON，可以通过 API 或 SDK 动态创建、修改和启动。LLM 可以生成工作流定义，Conductor 无需预注册即可立即执行。

## Conductor 支持人在回路（human-in-the-loop）吗？

支持。HUMAN 任务类型会暂停工作流执行，直到通过 API 收到外部信号（批准、拒绝或数据输入）。暂停可以跨服务器重启和部署存活。

## 支持哪些持久化后端？

Redis、PostgreSQL、MySQL、Cassandra 和 SQLite。根据你的规模和运维需求选择。

## 支持哪些消息代理？

Kafka、NATS、NATS Streaming、AMQP（RabbitMQ）、SQS，以及 Conductor 的内部队列。用它们实现事件驱动的工作流和外部系统集成。

## 如何让任务在一段时间（例如 1 小时、1 天）后才被放入队列？

轮询到任务后，把任务状态更新为 `IN_PROGRESS`，并将 `callbackAfterSeconds` 的值设置为期望的时间。该任务会一直留在队列中，直到指定的秒数过去，之后轮询它的工作者才会再次收到它。

如果任务配置了超时，且 `callbackAfterSeconds` 超过了超时值，任务会变为 TIMED_OUT。

## 一个工作流可以保持运行状态多久？我可以有一个持续运行数天甚至数月的工作流吗？

可以。只要任务的超时设置足以支撑长时间运行的工作流，它就会保持在运行状态。

## 我的工作流启动失败，报缺少任务的错误

确保所有任务都通过 `/metadata/taskdefs` API 注册了。添加缺失的任务定义（错误信息中会报告哪个），然后重试。

## 我的工作者在哪里运行？Conductor 如何运行我的任务？

Conductor 不运行工作者。当一个任务被调度时，它会放入由 Conductor 维护的队列中。工作者必须以固定间隔轮询任务（使用 `/tasks/poll` API），执行任务的业务逻辑，并通过 `POST {{ api_prefix }}/tasks` API 调用上报结果。
不过，Conductor 会在 Conductor 服务器上运行 [系统任务](../documentation/configuration/workflowdef/systemtasks/index.md)。

## 如何调度工作流在特定时间运行？

使用 Conductor 内置的调度器，将一个 Spring cron 表达式绑定到一个工作流启动请求上。你可以通过 [工作流调度指南](how-tos/Workflows/scheduling-workflows.md) 或 [调度器 API](../documentation/api/scheduler.md) 来创建、暂停、恢复、预览和检查调度。如果你要基于消息而非基于时间启动，请使用 [事件编排](how-tos/event-bus.md)。

## 我能用 Ruby / Go / Python / JavaScript / C# / Rust 配合 Conductor 吗？

可以。只要工作者能通过 HTTP 端点轮询并更新任务结果，就可以用任意语言编写。Conductor 为许多语言提供了官方和社区 SDK：

- **Java** — [conductor-oss/java-sdk](https://github.com/conductor-oss/java-sdk)
- **Python** — [conductor-oss/python-sdk](https://github.com/conductor-oss/python-sdk)
- **Go** — [conductor-oss/go-sdk](https://github.com/conductor-oss/go-sdk)
- **JavaScript** — [conductor-oss/javascript-sdk](https://github.com/conductor-oss/javascript-sdk)
- **C#** — [conductor-oss/csharp-sdk](https://github.com/conductor-oss/csharp-sdk)
- **Ruby** — [conductor-oss/ruby-sdk](https://github.com/conductor-oss/ruby-sdk)
- **Rust** — [conductor-oss/rust-sdk](https://github.com/conductor-oss/rust-sdk)

## 同一个任务被调度了两次，都显示 "attempt 0"。这是什么原因？

这几乎总是因为在未启用分布式锁的情况下运行了多个 Conductor 服务器实例。当锁关闭时，两个服务器实例可能各自领取同一个工作流，并独立地调度同一个任务——产生两条完全相同的记录，都位于 attempt 0，且彼此互不知情。

**修复方法**：启用分布式锁，使同一时间只有一个服务器处理给定工作流：

```properties
conductor.app.workflowExecutionLockEnabled=true
conductor.workflow-execution-lock.type=redis   # or zookeeper
```

完整配置请参见 [锁](running/deploy.md#locking)，包括 Redis 和 Zookeeper 选项。

如果你只运行单个服务器实例，则更可能的原因是 sweeper 和某个事件或回调同时对同一个工作流触发了 `decide`。上面的锁配置同样能解决这种情况。

## 我的工作流在运行，任务是 SCHEDULED 状态，但没有被处理。

确保工作者正在积极轮询该任务。打开 Conductor UI 的 `Task Queues` 标签，在搜索框中选择你的任务名。确保该任务的 `Last Poll Time` 是最新的。

在 Conductor 3.x 中，```conductor.redis.availabilityZone``` 默认为 ```us-east-1c```。确保它与你的工作者所在位置一致，并且与```conductor.redis.hosts```也一致。

## 如何配置工作流完成或失败时的通知？

当工作流失败时，你可以使用```failureWorkflow```参数配置一个"失败工作流"来运行。默认会传递三个参数：

* reason
* workflowId：用它来拉取失败工作流的详细信息。
* failureStatus

你还可以使用 Workflow Status Listener：

* 将工作流定义中的 workflowStatusListenerEnabled 字段设置为 true，以启用 [通知](../documentation/configuration/workflowdef/index.md#workflow-status-listener)。
* 添加一个自定义的 Workflow Status Listener 实现。参见 [Workflow Status Listener 扩展指南](../documentation/advanced/extend.md#workflow-status-listener)。
* 该通知的实现方式可以是：向外部系统发送通知，或者如 [事件处理器文档](../documentation/configuration/eventhandlers.md) 中所述，在 conductor 队列上发送事件以完成/失败另一个工作流中的另一个任务。

参考这份 [文档](../documentation/configuration/workflowdef/index.md#workflow-status-listener) 来扩展 conductor，使工作流完成/失败时发出事件/通知。

## 我希望工作者在进程被终止时停止轮询和执行任务。（Java 客户端）

在你的应用的 `PreDestroy` 块中，调用你创建的 `TaskRunnerConfigurer` 实例上的 `shutdown()` 方法，以便在进程被终止时对工作者进行优雅关停。

## 我能让任务提前退出，而不执行任务定义中配置的自动重试吗？

在工作者的 TaskResult 对象中将状态设置为 `FAILED_WITH_TERMINAL_ERROR`。这会把任务标记为 FAILED 并使工作流失败，同时不再重试该任务，作为一种快速失败机制。
