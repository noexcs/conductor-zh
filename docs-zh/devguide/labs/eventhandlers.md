---
description: "Event Handlers 实验 — 通过事件处理器发布事件并触发 Conductor 工作流的上手教程。"
---
# 事件与事件处理器

在本练习中，我们将：

* 使用 `Event` 任务向 Conductor 发布一个事件。
* 订阅事件并执行动作：
    * 启动一个工作流
    * 完成一个任务

Conductor 通过两个接口支持事件机制：

* [Event 任务](../../documentation/configuration/workflowdef/systemtasks/event-task.md)
* [事件处理器](../../documentation/configuration/eventhandlers.md)

## 创建工作流定义

我们来创建两个工作流：

* `test_workflow_for_eventHandler`：包含一个 `Event` 任务用于启动另一个工作流，以及一个由事件完成的 `WAIT` 系统任务。
* `test_workflow_startedBy_eventHandler`：包含一个 `Event` 任务，用于发布事件以完成上一个工作流中的 `WAIT` 任务。

向 `/metadata/workflow` 端点发送 `POST` 请求，payload 如下：

```json
{
  "name": "test_workflow_for_eventHandler",
  "description": "A test workflow to start another workflow with EventHandler",
  "version": 1,
  "tasks": [
    {
      "name": "test_start_workflow_event",
      "taskReferenceName": "start_workflow_with_event",
      "type": "EVENT",
      "sink": "conductor"
    },
    {
      "name": "test_task_tobe_completed_by_eventHandler",
      "taskReferenceName": "test_task_tobe_completed_by_eventHandler",
      "type": "WAIT"
    }
  ]
}
```

```json
{
  "name": "test_workflow_startedBy_eventHandler",
  "description": "A test workflow which is started by EventHandler, and then goes on to complete task in another workflow.",
  "version": 1,
  "tasks": [
    {
      "name": "test_complete_task_event",
      "taskReferenceName": "complete_task_with_event",
      "inputParameters": {
        "sourceWorkflowId": "${workflow.input.sourceWorkflowId}"
      },
      "type": "EVENT",
      "sink": "conductor"
    }
  ]
}
```

### 工作流中的 Event 任务

`EVENT` 任务是一种系统任务，我们在定义工作流时像定义其他任务一样定义它，并带 `sink` 参数。另外，`EVENT` 任务无需在使用前先注册。`WAIT` 任务也是如此。
因此，我们不会为这些工作流注册任何任务。

### 事件已发出，但尚未被处理

当你尝试启动 `test_workflow_for_eventHandler` 工作流时会发现：事件成功发出，但第二个工作流 `test_workflow_startedBy_eventHandler` 并没有被启动。我们已经发出了事件，但还需要为 Conductor 定义 `Event Handlers`（事件处理器），让它基于事件执行相应的 `actions`（动作）。下面我们来创建事件处理器。

## 创建事件处理器

事件处理器的定义与任务或工作流定义大同小异。我们先从 name 开始：

```json
{
  "name": "test_start_workflow"
}
```

事件处理器需要知道自己要监听哪个队列，这通过 `event` 参数定义。

使用 Conductor 队列时，`event` 的格式为：

```conductor:{workflow_name}:{taskReferenceName}```

使用 SQS 时，格式为：

```sqs:{my_sqs_queue_name}```

```json
{
  "name": "test_start_workflow",
  "event": "conductor:test_workflow_for_eventHandler:start_workflow_with_event"
}
```

事件处理器可以对特定的 `event` 队列执行 `actions` 数组参数中定义的一组动作。

```json
{
  "name": "test_start_workflow",
  "event": "conductor:test_workflow_for_eventHandler:start_workflow_with_event",
  "actions": [
      "<insert-actions-here>"
  ],
  "active": true
}
```

下面定义 `start_workflow` 动作。我们需要传入要启动的工作流名称。`start_workflow` 参数可以使用通用的[启动工作流请求](../../documentation/api/startworkflow.md)中的任意字段。这里我们传入 workflowId，以便后续的 Complete Task 事件处理器使用。

```json
{
    "action": "start_workflow",
    "start_workflow": {
        "name": "test_workflow_startedBy_eventHandler",
        "input": {
            "sourceWorkflowId": "${workflowInstanceId}"
        }
    }
}
```

向 `/event` 端点发送 `POST` 请求：

```json
{
  "name": "test_start_workflow",
  "event": "conductor:test_workflow_for_eventHandler:start_workflow_with_event",
  "actions": [
    {
      "action": "start_workflow",
      "start_workflow": {
        "name": "test_workflow_startedBy_eventHandler",
        "input": {
          "sourceWorkflowId": "${workflowInstanceId}"
        }
      }
    }
  ],
  "active": true
}
```

类似地，再创建另一个用于完成任务的事件处理器。

```json
{
  "name": "test_complete_task_event",
  "event": "conductor:test_workflow_startedBy_eventHandler:complete_task_with_event",
  "actions": [
    {
    	"action": "complete_task",
    	"complete_task": {
	        "workflowId": "${sourceWorkflowId}",
	        "taskRefName": "test_task_tobe_completed_by_eventHandler"
	     }
    }
  ],
  "active": true
}
```

## 小结

把以上内容全部配置好后，启动 `test_workflow_for_eventHandler` 应当：

1. 启动 `test_workflow_startedBy_eventHandler` 工作流。
2. 将 `test_task_tobe_completed_by_eventHandler` WAIT 任务置为 `IN_PROGRESS`。
3. `test_workflow_startedBy_eventHandler` 的 event 任务会发布一个事件，以完成上面的 WAIT 任务。
4. 两个工作流都会进入 `COMPLETED` 状态。
