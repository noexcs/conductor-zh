# Workflow Message Queue（WMQ）

**一句话概览** — 每个工作流现在都有一个队列。你可以用这个队列把工作流变成一个事件循环：它空闲地等待消息，处理每一条消息，然后再回到等待状态。

## 工作原理

WMQ 为每个运行中的 Conductor 工作流添加了一个持久化消息队列。在工作流活动期间，你可以从任何地方向其中推送消息——另一个服务、Kafka 消费者、webhook 处理器、甚至一个人——工作流会接收这些消息并据此行动。

两个组件使这一切成为可能：

1. **`POST /api/workflow/{workflowId}/messages`** — 由 Conductor 暴露的一个 HTTP 端点，接受 JSON 载荷并将其放入该工作流的队列。
2. **`PULL_WORKFLOW_MESSAGES`** — 一种新的 Conductor 系统任务，它会阻塞直到消息到达，然后以 `output.messages` 携带该批消息完成。

## 前置条件

WMQ 默认是禁用的。在注册使用 `PULL_WORKFLOW_MESSAGES` 的工作流或调用推送端点之前，请先在 Conductor 服务器上启用它：

```properties
conductor.workflow-message-queue.enabled=true
```

当该属性为 `false` 时，Conductor 不会注册该系统任务或 HTTP 端点；端点会返回 `404 Not Found`。

## 使用 WMQ

在你的工作流定义中添加一个 `PULL_WORKFLOW_MESSAGES` 任务：

```json
{
  "name": "wait_for_message",
  "taskReferenceName": "wait_for_message_ref",
  "type": "PULL_WORKFLOW_MESSAGES",
  "inputParameters": {
    "batchSize": 1
  }
}
```

然后向其中推送消息：

```bash
curl -X POST http://localhost:8080/api/workflow/{workflowId}/messages \
  -H "Content-Type: application/json" \
  -d '{"text": "hello"}'
```

该任务以如下方式完成：

```json
{
  "messages": [
    {
      "id": "3f2504e0-4f89-11d3-9a0c-0305e82c3301",
      "workflowId": "8e2c14e1-...",
      "payload": { "text": "hello" },
      "receivedAt": "2025-06-15T10:30:00Z"
    }
  ],
  "count": 1
}
```

你的工作流通过 `output.messages[0].payload` 访问用户数据。`id` 和 `receivedAt` 字段由 Conductor 在接收消息时添加。

**推送错误：**
- `404 Not Found` — 工作流 ID 不存在，或 WMQ 功能未启用。
- `409 Conflict` — 工作流不处于 `RUNNING` 状态（已完成、失败、已终止等）。消息不会被保存。
- `429 Too Many Requests` — 队列已满（达到 `maxQueueSize`）。调用方必须退避并重试。

### 事件循环模式

对于需要处理无界消息流的工作流，将该任务包裹在 `DO_WHILE` 中：

```json
{
  "name": "message_loop",
  "taskReferenceName": "message_loop_ref",
  "type": "DO_WHILE",
  "loopCondition": "$.message_loop_ref['iteration'] < 100",
  "loopOver": [
    {
      "name": "pull_message",
      "taskReferenceName": "pull_message_ref",
      "type": "PULL_WORKFLOW_MESSAGES",
      "inputParameters": { "batchSize": 1 }
    },
    {
      "name": "process_message",
      "taskReferenceName": "process_message_ref",
      "type": "INLINE",
      "inputParameters": {
        "evaluatorType": "javascript",
        "expression": "function e() { return { payload: $.messages[0].payload }; } e();",
        "messages": "${pull_message_ref.output.messages}"
      }
    }
  ]
}
```

循环会停留在 `PULL_WORKFLOW_MESSAGES` 上，直到下一条消息到达。

## 将 WMQ 与智能体一起使用

WMQ 与框架无关。在 Conductor 图中使用 `PULL_WORKFLOW_MESSAGES`，让执行停留在原地直到消息到达，然后将返回的载荷传给下一个任务。对于由 SDK 编写的智能体，请参阅 [Conductor 智能体](../devguide/ai/conductor-agents.md)，并将特定于框架的运行时代码保留在其对应的 SDK 示例中维护。

### Kafka 桥接示例

该模式同样可以作为外部事件流的桥接。Kafka 消费者可以将每条记录转换为一个 `POST /api/workflow/{workflowId}/messages` 请求，使用上文所示的载荷结构。该消费者的实现应保留在其所属的 SDK 或服务仓库中；它与工作流的智能体步骤所使用的框架无关。

## 配置

```properties
conductor.workflow-message-queue.enabled=true
conductor.workflow-message-queue.maxQueueSize=1000
conductor.workflow-message-queue.ttlSeconds=86400
conductor.workflow-message-queue.maxBatchSize=100
```

| 属性 | 默认值 | 说明 |
|---|---|---|
| `enabled` | `false` | 启用 WMQ 功能 |
| `maxQueueSize` | `1000` | 每个工作流可排队的最大消息数 |
| `ttlSeconds` | `86400` | 消息 TTL（24 小时） |
| `maxBatchSize` | `100` | 每次 `PULL_WORKFLOW_MESSAGES` 轮询返回的最大消息数 |
