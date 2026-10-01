---
description: "Start Workflow 任务 — 从正在运行的工作流内部异步启动新的 Conductor 工作流执行。"
---
# Start Workflow
```json
"type" : "START_WORKFLOW"
```

Start Workflow 任务（`START_WORKFLOW`）从当前工作流启动另一个工作流。与 [Sub Workflow](sub-workflow-task.md) 任务不同，由 Start Workflow 任务触发的工作流将异步执行。也就是说，当前工作流不会等待被启动的工作流完成，而是继续执行下一个任务。

当请求的工作流进入 RUNNING 状态时，Start Workflow 任务即被标记为 COMPLETED，无论其最终状态如何。

## 任务参数

在 Start Workflow 任务配置的 `inputParameters` 内部使用以下参数。

| 参数          | 类型                | 描述                                       | 必填 / 可选  |
| ------------------ | ------------------- | ------------------------------------------------- | -------------------- |
| startWorkflow | Map[String, Any] | 包含所请求工作流配置的映射，例如名称和版本。有关此参数应包含的内容，请参阅 [Start Workflow API](../../../api/startworkflow.md#request-body)。 | 必填。 |

## 任务配置
以下是 Start Workflow 任务的任务配置。

```json
{
  "name": "start_workflow",
  "taskReferenceName": "start_workflow_ref",
  "inputParameters": {
    "startWorkflow": {
      "name": "someName",
      "input": {
        "someParameter": "someValue",
        "anotherParameter": "anotherValue"
      },
      "version": 1,
      "correlationId": ""
    }
  },
  "type": "START_WORKFLOW"
}
```

## 输出


Start Workflow 任务将返回以下参数。

| 名称             | 类型         | 描述                                                   |
| ---------------- | ------------ | ------------------------------------------------------------- |
| workflowId | String | 被启动工作流的工作流执行 ID。 |


## 局限性

由于 Start Workflow 任务既不会等待被启动的工作流完成，也不会传回其输出，因此无法从当前工作流访问被启动工作流的输出。如有需要，可以改用 [Sub Workflow](sub-workflow-task.md) 任务。
