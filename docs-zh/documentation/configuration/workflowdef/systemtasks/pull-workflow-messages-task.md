---
description: "PULL_WORKFLOW_MESSAGES 系统任务 — 等待并拉取 Conductor 工作流消息队列中的消息。"
---

# Pull Workflow Messages 任务

```json
"type": "PULL_WORKFLOW_MESSAGES"
```

`PULL_WORKFLOW_MESSAGES` 等待工作流消息队列中可用的消息，并将接收到的批次提供给工作流。它面向使用工作流消息队列特性而非工作者轮询器的工作流。

## 可用性

仅当 `conductor.workflow-message-queue.enabled=true` 时才会注册此任务。它还需要相应的工作流消息队列基础设施和配置。如果该特性被禁用，使用此类型的工作流将无法被映射。

## 配置

映射器会解析任务的 `inputParameters`，队列工作者会消费它们。当工作流需要限制单次拉取数量时，提供 `batchSize`，并附带已配置的消息队列实现所需的任何队列特定输入。

```json
{
  "name": "pull_messages",
  "taskReferenceName": "pull_messages",
  "type": "PULL_WORKFLOW_MESSAGES",
  "inputParameters": {
    "batchSize": 10
  }
}
```

任务将保持进行中，直到消息可用。特性配置和投递语义请参阅[工作流消息队列](../../../../wmq/workflow-message-queue.md)。
