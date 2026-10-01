---
description: "概览 Conductor 内置的系统任务 — HTTP、Event、Human、Wait、Inline、Kafka Publish、JSON JQ Transform、LLM 编排、MCP 函数调用等，用于持久化工作流编排。"
---

# 系统任务

系统任务是运行在 Conductor 服务器上的内置任务。它们无需外部工作者即可执行，让你能够开箱即用地使用常见操作构建工作流。

## 可用系统任务

| 系统任务 | 类型 | 说明 |
| :--- | :--- | :--- |
| [HTTP](http-task.md) | `HTTP` | 调用任意 HTTP/REST 端点。支持带请求头、请求体以及连接/读取超时的 GET、POST、PUT、DELETE。 |
| [Inline](inline-task.md) | `INLINE` | 在服务器端执行轻量级 JavaScript 或 GraalVM Python 表达式。适用于数据转换、校验和简单逻辑。 |
| [Event](event-task.md) | `EVENT` | 向外部系统发布事件 — Kafka、NATS、NATS Streaming、AMQP（RabbitMQ）、SQS 或 Conductor 内部队列。 |
| [Wait](wait-task.md) | `WAIT` | 暂停工作流执行，直到指定时间、时长或外部信号到来。 |
| [Human](human-task.md) | `HUMAN` | 等待外部信号，通常是人工审批或手动操作。任务保持 `IN_PROGRESS` 状态，直到通过 API 完成。 |
| [Kafka Publish](kafka-publish-task.md) | `KAFKA_PUBLISH` | 直接发布消息到 Kafka 主题，支持可配置的序列化器和请求头。 |
| [JSON JQ Transform](json-jq-transform-task.md) | `JSON_JQ_TRANSFORM` | 使用 [jq](https://jqlang.org/) 表达式转换 JSON 数据。在数据重塑、过滤和聚合方面功能强大。 |
| [No Op](noop-task.md) | `NOOP` | 什么都不做。可用作占位符，或在 fork/join 模式中合并分支。 |
| [JDBC](jdbc-task.md) | `JDBC` | 针对关系型数据库（MySQL、PostgreSQL、Oracle 等）执行 SQL 查询和更新，支持连接池和事务管理。 |
| [Pull Workflow Messages](pull-workflow-messages-task.md) | `PULL_WORKFLOW_MESSAGES` | 从工作流消息队列中拉取一批消息；需要 `conductor.workflow-message-queue.enabled=true`。 |

## 操作符（流程控制）

这些同样是系统任务，但它们控制工作流的执行流程，而不是执行具体工作：

| 操作符 | 类型 | 说明 |
| :--- | :--- | :--- |
| [Fork/Join](../operators/fork-task.md) | `FORK_JOIN` | 在并行分支中执行任务，然后汇合。 |
| [Dynamic Fork](../operators/dynamic-fork-task.md) | `FORK_JOIN_DYNAMIC` | 在运行时动态创建并行分支。 |
| [Join](../operators/join-task.md) | `JOIN` | 等待并行分支完成。 |
| [Exclusive Join](../operators/exclusive-join-task.md) | `EXCLUSIVE_JOIN` | 当第一个选中的分支完成时继续执行。 |
| [Switch](../operators/switch-task.md) | `SWITCH` | 基于表达式或值进行条件分支。 |
| [Do While](../operators/do-while-task.md) | `DO_WHILE` | 循环执行任务，直到满足条件。 |
| [Sub Workflow](../operators/sub-workflow-task.md) | `SUB_WORKFLOW` | 将另一个工作流作为任务执行。 |
| [Start Workflow](../operators/start-workflow-task.md) | `START_WORKFLOW` | 异步启动另一个工作流（发后即忘）。 |
| [Set Variable](../operators/set-variable-task.md) | `SET_VARIABLE` | 设置或更新工作流级别的变量。 |
| [Terminate](../operators/terminate-task.md) | `TERMINATE` | 以指定状态终止工作流。 |
| [Dynamic](../operators/dynamic-task.md) | `DYNAMIC` | 在运行时确定要执行的任务类型。 |

## AI 任务

[AI 任务](ai-tasks.md) 是 LLM、向量/嵌入、媒体/PDF、MCP 和 A2A 任务族的完整目录。它们需要 `conductor.integrations.ai.enabled=true` 以及相应的提供方特定配置。

## 已弃用

| 任务 | 替代方案 |
| :--- | :--- |
| Lambda | 请改用 [Inline](inline-task.md)。 |
| Decision | 请改用 [Switch](../operators/switch-task.md)。 |
