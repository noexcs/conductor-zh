---
description: "为你的 Conductor 工作流选择合适的任务类型——用于微服务编排和工作流自动化的系统任务、操作符和工作者任务。"
---

# 选择任务

任务是 Conductor 工作流的构建块。在本指南中，了解 Conductor OSS 中可用的任务以及它们之间的差异。

## 内置任务

内置任务让你无需构建和部署自己的任务工作者，即可在 Conductor 服务器上轻松运行常见任务。以下是 Conductor 中可用的内置任务介绍：

* **[系统任务](../../../documentation/configuration/workflowdef/systemtasks/index.md)** 允许你无需自定义工作者即可快速上手的常见任务。
* **[操作符](../../../documentation/configuration/workflowdef/operators/index.md)** 让你以极少的代码声明式设计工作流的控制流和逻辑。

### 系统任务

以下是 Conductor OSS 中供常用用途的系统任务：

| 系统任务                  | 描述                          |
| :-------------------- | :----------------------------------- |
| [Event](../../../documentation/configuration/workflowdef/systemtasks/event-task.md)       | 向外部事件系统（AMQP、SQS、Kafka 等）发布事件。              |
| [HTTP](../../../documentation/configuration/workflowdef/systemtasks/http-task.md)         | 调用 API 或 HTTP 端点。                                 |
| [Human](../../../documentation/configuration/workflowdef/systemtasks/human-task.md)       | 等待外部信号。                                  |
| [Inline](../../../documentation/configuration/workflowdef/systemtasks/inline-task.md)     | 内联执行轻量级 JavaScript 代码。                   |
| [No Op](../../../documentation/configuration/workflowdef/systemtasks/noop-task.md)        | 什么都不做。                                                   |
| [JSON JQ Transform](../../../documentation/configuration/workflowdef/systemtasks/json-jq-transform-task.md) | 使用 jq 清理或转换 JSON 数据。      |
| [Kafka Publish](../../../documentation/configuration/workflowdef/systemtasks/kafka-publish-task.md)  | 向 Kafka 发布消息。                         |
| [Wait](../../../documentation/configuration/workflowdef/systemtasks/wait-task.md)         | 等待直到设定的时间或时长过去。                 |


### 操作符

以下是 Conductor OSS 中用于管理执行流的操作符：

| 操作符                        | 描述         |
| -------------------------- | ----------------------------------------- |
| [Do While](../../../documentation/configuration/workflowdef/operators/do-while-task.md)         | 重复执行任务，类似 _do…while…_ 语句。     |
| [Dynamic](../../../documentation/configuration/workflowdef/operators/dynamic-task.md)           | 动态执行任务，类似函数指针。           |
| [Dynamic Fork](../../../documentation/configuration/workflowdef/operators/dynamic-fork-task.md) | 并行执行动态数量的任务。 |
| [Fork](../../../documentation/configuration/workflowdef/operators/fork-task.md)                 | 并行执行静态数量的任务。  |
| [Join](../../../documentation/configuration/workflowdef/operators/join-task.md)                 | 在 Fork 或 Dynamic Fork 之后汇合各分支，然后再继续下一个任务。                        |
| [Set Variable](../../../documentation/configuration/workflowdef/operators/set-variable-task.md)     | 创建或更新工作流变量。        |
| [Start Workflow](../../../documentation/configuration/workflowdef/operators/start-workflow-task.md) | 异步启动另一个工作流，类似入口点。   |
| [Sub Workflow](../../../documentation/configuration/workflowdef/operators/sub-workflow-task.md) | 同步启动另一个工作流，类似子程序。  |
| [Switch](../../../documentation/configuration/workflowdef/operators/switch-task.md)             | 有条件地执行任务，类似 _if…else…_ 语句。     |
| [Terminate](../../../documentation/configuration/workflowdef/operators/terminate-task.md)       | 终止当前工作流，类似 _return_ 语句。                       |

## 自定义任务

如果你需要实现超出 Conductor 系统任务范围的自定义逻辑，可以改用 Worker（`SIMPLE`）任务。与内置任务不同，Worker 任务需要在 Conductor 环境之外设置一个工作者，由它轮询并执行该任务。

## 任务对比

为帮助你决定使用哪些任务，这里详细对比了 Conductor 中功能相似的任务。

### Inline 任务与 Worker 任务

[Inline 任务](../../../documentation/configuration/workflowdef/systemtasks/inline-task.md)用于直接在工作流内部执行自定义 JavaScript 代码。它非常适合**简单数据转换、条件判断或小型计算**等轻量级操作。由于代码在 Conductor JVM 内执行，Inline 任务具有低延迟、无网络开销和更易调试的优势。但它在其他语言、自定义库、框架或技术栈的使用上也存在限制。


Worker 任务由外部任务工作者处理，这些工作者执行自定义函数或服务
是工作流中执行特定任务的外部自定义函数或服务。可以用任意选择的语言（Python、Java 等）编写，可以执行**复杂业务逻辑、自定义算法或长时间运行的操作**。Worker 任务在 Conductor 服务器之外运行，这意味着需要额外的基础设施搭建和日志机制。

### Event 任务与 Kafka Publish 任务

如果你只需要向 Kafka 主题发布消息供外部服务使用，[Kafka Publish](../../../documentation/configuration/workflowdef/systemtasks/kafka-publish-task.md) 任务更简单。

相比之下，[Event](../../../documentation/configuration/workflowdef/systemtasks/event-task.md) 任务支持更复杂的设置，例如使用事件启动 Conductor 工作流，或让 Conductor 消费消息。它还支持更广泛的事件 broker，涵盖 AMQP、NATS、SQS、Kafka 以及 Conductor 自己的内部队列。


### Wait 任务与 Human 任务

[Wait](../../../documentation/configuration/workflowdef/systemtasks/wait-task.md) 任务和 [Human](../../../documentation/configuration/workflowdef/systemtasks/human-task.md) 任务都支持等待直到满足特定条件。当工作流需要等待特定时长或时间戳时使用 Wait 任务，当工作流需要等待外部触发时使用 Human 任务。

### Start Workflow 任务与 Sub Workflow 任务

[Start Workflow](../../../documentation/configuration/workflowdef/operators/start-workflow-task.md) 和 [Sub Workflow](../../../documentation/configuration/workflowdef/operators/sub-workflow-task.md) 任务都适用于在工作流内启动另一个工作流。不同之处在于：Start Workflow 任务启动另一个工作流后不等待其完成就继续下一个任务，而 Sub Workflow 任务会等待子工作流到达终态后才继续下一个任务。

Sub Workflow 任务在父工作流与子工作流之间提供了更紧的耦合。这在需要将工作流进度和状态关联起来，或需要把子工作流的输出传回父工作流的场景中很有用。


### Fork 任务与 Dynamic Fork 任务

[Fork](../../../documentation/configuration/workflowdef/operators/fork-task.md) 和 [Dynamic Fork](../../../documentation/configuration/workflowdef/operators/dynamic-fork-task.md) 都促进任务的并行执行。Fork 任务执行预定数量的分支，而 Dynamic Fork 在运行时执行可变数量的分支。

如果每个分支必须运行不同的任务集合，最好使用 Fork 任务，因为 Dynamic Fork 的所有分支只能运行同一个任务。


### Dynamic 任务与 Switch 任务

当要运行的具体任务只在运行时才能确定时，[Switch](../../../documentation/configuration/workflowdef/operators/switch-task.md) 任务和 [Dynamic](../../../documentation/configuration/workflowdef/operators/dynamic-task.md) 任务都很有用。使用 Switch 任务可以轻松预定义并设置每个 switch 分支的具体条件；使用 Dynamic 任务则可以在工作流中标记一个动态点，而无需事先把所有分支选项都预设进工作流定义。

在工作流图中，Dynamic 任务会呈现更简化的视图，因为它只显示所选的任务。而 Switch 任务会呈现更全面的视图，展示工作流可能走过的所有路径。

以下是一些在 Dynamic 任务和 Switch 任务之间做决定的场景：


| 场景                        | 应使用的任务         |
| -------------------------- | ----------------------------------------- |
| 分支选项数量极多，或具体选项尚未确定。         | Dynamic    |
| 需要一个默认分支选项。       | Switch    |
| 每个分支选项涉及多个任务。       | Switch    |
| 每个 switch 分支的条件相对直接。         | Switch    |
| 每个 switch 分支的条件不断变化，或需要更复杂的逻辑。      | Dynamic    |

如果你选择 Dynamic 任务，必须搭建控制流，以便在运行时确定要运行的任务。例如，使用一个前置任务，把任务名传给 Dynamic 任务。
