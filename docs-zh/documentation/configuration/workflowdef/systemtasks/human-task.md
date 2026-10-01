---
description: "在 Conductor 中配置 Human 任务，暂停工作流以等待人工审批或外部信号。支持人在回路（human-in-the-loop）和智能体式（agentic）工作流模式。"
---

# Human 任务
```json
"type" : "HUMAN"
```

Human 任务（`HUMAN`）用于暂停工作流并等待外部信号。它作为一个门控，保持 IN_PROGRESS 状态，直到被外部触发器标记为 COMPLETED 或 FAILED。

当工作流需要暂停并等待人工介入（例如人工审批）时，可以使用 Human 任务。它也可以与来自外部源的事件配合使用，例如 Kafka、SQS 或 Conductor 的内部队列机制。

## 任务参数

配置 Human 任务不需要任何参数。

## JSON 配置

以下是 Human 任务的任务配置。

```json
{
	"name": "human",
  "taskReferenceName": "human_ref",
	"inputParameters": {},
	"type": "HUMAN"
}
```

## 完成 Human 任务

完成 Human 任务有几种方式：

- 使用任务更新 API
- 使用事件处理器


### 任务更新 API
使用任务更新 API（`POST api/tasks`）完成 Human 任务。提供 `taskId`、任务状态和期望的任务输出。

使用 CLI：

```bash
conductor task update-execution --workflow-id {workflowId} --task-ref-name waiting_around_ref --status COMPLETED --output '{"data_key":"somedatatoWait1","data_key2":"somedatatoWAit2"}'
```

### 事件处理器
如果启用了 SQS 集成，也可以使用队列更新 API 解决 Human 任务：

1. `POST api/queue/update/{workflowId}/{taskRefName}/{status}`
2. `POST api/queue/update/{workflowId}/task/{taskId}/{status}`

POST 消息体中发送的任何参数都会作为任务的输出重复。例如，如果我们发送如下 COMPLETED 消息：

??? note "使用 cURL"
    ```bash
    curl -X "POST" "{{ server_host }}{{ api_prefix }}/queue/update/{workflowId}/waiting_around_ref/COMPLETED" \
      -H 'Content-Type: application/json' \
      -d '{"data_key":"somedatatoWait1","data_key2":"somedatatoWAit2"}'
    ```

Human 任务的输出将是：

```json
{
  "data_key":"somedatatoWait1",
  "data_key2":"somedatatoWAit2"
}
```


或者，也可以配置一个使用 `complete_task` 动作的[事件处理器](../../eventhandlers.md)。

## 监控 Human 任务：获取回调和通知

当工作流到达 Human 任务时，你可能希望收到通知或回调以触发下一个动作（例如发送邮件、通知 Slack 频道或触发外部系统）。以下是推荐的模式：

### 模式 1：轮询工作流状态 API

最简单的方法是轮询工作流执行状态，检查处于 `IN_PROGRESS` 状态的 Human 任务：

```bash
# Get workflow execution status
curl '{{ server_host }}/api/workflow/{workflowId}' \
  -H 'accept: application/json'
```

解析响应以找到 `taskType: "HUMAN"` 且 `status: "IN_PROGRESS"` 的任务。

**优点：** 实现简单，无需额外配置
**缺点：** 需要轮询，非实时

### 模式 2：使用 Conductor 内部事件的事件处理器

当任务状态发生变化时，Conductor 可以发布内部事件。你可以配置事件处理器来监听这些事件：

```json
{
  "name": "human_task_notification_handler",
  "event": "conductor:TASK_STATUS_CHANGE",
  "condition": "$.taskType == 'HUMAN' && $.status == 'IN_PROGRESS'",
  "actions": [
    {
      "action": "start_workflow",
      "start_workflow": {
        "name": "notification_workflow",
        "input": {
          "workflowId": "${workflowId}",
          "taskRefName": "${taskRefName}",
          "taskStatus": "${status}"
        }
      }
    }
  ]
}
```

每当 Human 任务进入 `IN_PROGRESS` 状态时，这会触发一个通知工作流。

### 模式 3：通过 Event 任务进行 Webhook 集成

在 Human 任务之前添加一个 EVENT 任务以发送 webhook 通知：

```json
{
  "name": "notify_human_task",
  "taskReferenceName": "notify_ref",
  "type": "EVENT",
  "sink": "kafka:human-task-notifications",
  "inputParameters": {
    "workflowId": "${workflow.input.workflowId}",
    "taskRefName": "human_ref",
    "eventType": "HUMAN_TASK_PENDING"
  }
},
{
  "name": "human_approval",
  "taskReferenceName": "human_ref",
  "type": "HUMAN"
}
```

然后配置事件处理器或外部消费者来处理这些通知。

### 模式 4：带回调输出的任务完成

完成 Human 任务时，在输出中包含回调信息：

```bash
curl -X POST "{{ server_host }}/api/tasks" \
  -H 'Content-Type: application/json' \
  -d '{
    "taskId": "${taskId}",
    "status": "COMPLETED",
    "output": {
      "approvedBy": "user@example.com",
      "approvedAt": "2026-04-22T10:30:00Z",
      "comments": "Approved for production deployment"
    }
  }'
```

该输出可供下游任务使用，可用于审计跟踪或进一步通知。

### 模式 5：外部系统集成

对于实时通知，请与外部系统集成：

1. **Slack/Teams**：使用事件处理器触发一个通知工作流，向 Slack/Teams webhook 发送消息
2. **邮件**：通过 SMTP 或邮件服务 API 发送邮件通知
3. **短信/推送**：集成 Twilio、Pushover 或类似服务
4. **自定义 Webhook**：当 Human 任务处于待定状态时，向你的内部系统发起 POST

### 最佳实践

- **使用关联 ID**：在所有通知中包含 `workflowId` 和 `taskRefName`，便于跟踪
- **设置超时**：考虑添加超时逻辑，以升级处理长时间未审批的 Human 任务
- **审计跟踪**：记录所有 Human 任务的完成信息，包括时间戳和用户信息
- **幂等性**：确保通知处理器是幂等的，以处理重复事件

## 示例：完整通知流程

```json
{
  "name": "approval_workflow",
  "version": 1,
  "tasks": [
    {
      "name": "send_approval_request",
      "taskReferenceName": "send_request_ref",
      "type": "HTTP",
      "inputParameters": {
        "http_request": {
          "method": "POST",
          "url": "https://hooks.slack.com/services/xxx",
          "body": {
            "text": "Approval needed for workflow ${workflow.input.requestId}"
          }
        }
      }
    },
    {
      "name": "wait_for_approval",
      "taskReferenceName": "approval_ref",
      "type": "HUMAN"
    },
    {
      "name": "send_approval_result",
      "taskReferenceName": "send_result_ref",
      "type": "HTTP",
      "inputParameters": {
        "http_request": {
          "method": "POST",
          "url": "https://hooks.slack.com/services/xxx",
          "body": {
            "text": "Approval ${approval_ref.output.status} by ${approval_ref.output.approvedBy}"
          }
        }
      }
    }
  ]
}
```

此工作流：
1. 在需要审批时发送 Slack 通知
2. 等待人工审批
3. 发送包含审批结果的后续通知
