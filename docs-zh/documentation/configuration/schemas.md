---
description: "Conductor 工作流定义、任务定义、工作流执行与任务执行的规范 JSON Schema。"
---

# Schemas

Conductor 以 JSON Schema 文件的形式，发布其定义对象与运行时对象所对应的详细、带版本号的契约。这些 Schema 是字段级校验的权威依据；本页指出在集成设计时最实用的对象与关系。

## 定义对象 { #definition-objects .schema-table-heading }

| Schema | 用途 | 标识与重要关系 |
|---|---|---|
| [WorkflowDef.json](https://github.com/conductor-oss/conductor/blob/main/schemas/WorkflowDef.json) | 可复用的工作流蓝图。 | `name` 与 `version` 标识一个定义。`tasks` 包含 `WorkflowTask` 配置；`inputParameters`、`outputParameters`、超时、owner 以及失败工作流设置共同构成其契约。 |
| [TaskDef.json](https://github.com/conductor-oss/conductor/blob/main/schemas/TaskDef.json) | 为工作者（`SIMPLE`）任务类型注册的配置。 | `name` 标识任务定义。当工作流中的任务引用该类型时，重试策略、超时值、速率限制与并发设置生效。 |

## 运行时对象 { #runtime-objects .schema-table-heading }

| Schema | 用途 | 标识与生命周期关系 |
|---|---|---|
| [Workflow.json](https://github.com/conductor-oss/conductor/blob/main/schemas/Workflow.json) | 一次 `WorkflowDef` 的执行。 | `workflowId` 标识该次执行；`workflowName`、`workflowVersion`、`status`、时间戳、输入/输出、变量与 `tasks` 记录其生命周期。 |
| [Task.json](https://github.com/conductor-oss/conductor/blob/main/schemas/Task.json) | 工作流内部一个已调度或已执行的任务。 | `taskId` 标识运行时任务；`workflowInstanceId` 将其关联到所属工作流。`taskType`、`referenceTaskName`、状态、输入/输出以及重试/超时状态描述其执行。 |

定义对象通过 [Metadata API](../api/metadata.md) 提交。运行时对象由 [Workflow API](../api/workflow.md) 与 [Task API](../api/task.md) 返回。在生成客户端、校验载荷或查看完整字段列表时，请使用所链接的 Schema 文件。
