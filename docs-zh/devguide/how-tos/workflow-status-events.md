---
description: 把 Conductor 工作流生命周期事件发布到 Kafka、Conductor 队列或出站 HTTP Webhook。
---

# 工作流状态事件

`workflow-event-listener` 模块为在工作流定义中通过 `workflowStatusListenerEnabled: true` 选择加入的工作流发布生命周期通知。标准服务器包含该模块。用 `conductor.workflow-status-listener.type` 配置一个监听器，或使用复合监听器（composite listener）发布到多个目标。

```json
{
  "name": "order_processing",
  "version": 1,
  "workflowStatusListenerEnabled": true,
  "tasks": []
}
```

工作流状态事件是出站通知。它们不会注册入站 Webhook，也不会创建事件处理器；要接收和路由 broker 事件，请使用[事件编排](event-bus.md)。

## 选择发布者

| 类型 | 目标 | 事件 |
|---|---|---|
| `kafka` | Kafka 主题 | `STARTED`、`RERAN`、`RETRIED`、`PAUSED`、`RESUMED`、`RESTARTED`、`COMPLETED`、`TERMINATED`、`FINALIZED` |
| `queue_publisher` | Conductor 队列 | 完成、终止和终结（finalize）的摘要 |
| `workflow_publisher` | 出站 HTTP Webhook | 已配置的生命周期状态；默认为 `COMPLETED` 和 `TERMINATED` |
| `composite` | 多个发布者 | 所选发布者事件的合集 |

每个发布者都会序列化工作流摘要数据。Kafka 发布者把该摘要包装进一个带 `workflowName`、`eventType` 和 `payload` 的对象；它使用工作流 ID 作为 Kafka 记录 key。

## 发布到 Kafka

把监听器类型设置为 `kafka`。Kafka 生产者配置放在 `conductor.workflow-status-listener.kafka.producer` 之下；未配置默认主题时，监听器使用 `workflow-status-events`。

```properties
conductor.workflow-status-listener.type=kafka
conductor.workflow-status-listener.kafka.producer[bootstrap.servers]=kafka:29092
conductor.workflow-status-listener.kafka.default-topic=workflow-status-events
conductor.workflow-status-listener.kafka.event-topics.completed=workflow-completed-events
```

`event-topics` 按事件名覆盖默认主题。所配置的生产者映射仅限于受支持的 Kafka 生产者属性；需要时在其中配置序列化器、重试、确认（acknowledgements）和 TLS。

## 发布到 Conductor 队列

把监听器类型设置为 `queue_publisher`。完成、终止和终结会把序列化的 `WorkflowSummary` 发送到对应的队列。

```properties
conductor.workflow-status-listener.type=queue_publisher
conductor.workflow-status-listener.queue-publisher.successQueue=_callbackSuccessQueue
conductor.workflow-status-listener.queue-publisher.failureQueue=_callbackFailureQueue
conductor.workflow-status-listener.queue-publisher.finalizeQueue=_callbackFinalizeQueue
```

必须至少配置一个成功或失败队列。这些是 Conductor 任务队列，不是[事件编排](event-bus.md)中记载的事件处理器提供商队列。

## 发布到 HTTP Webhook

把监听器类型设置为 `workflow_publisher` 并配置通知 URL。发布者会异步地把工作流状态通知发送到该 URL。

```properties
conductor.workflow-status-listener.type=workflow_publisher
conductor.status-notifier.notification.url=https://example.internal/workflow-events
conductor.status-notifier.notification.subscribed-workflow-statuses=RUNNING,COMPLETED,TERMINATED
```

省略 `subscribed-workflow-statuses` 时，Webhook 发布者订阅 `COMPLETED` 和 `TERMINATED`。它还可以订阅 `RUNNING`、`PAUSED`、`RESUMED`、`RESTARTED`、`RETRIED`、`RERAN` 和 `FINALIZED`。

## 发布到多个目标

使用 `composite`，并附上 `kafka`、`queue_publisher`、`workflow_publisher` 和 `archive` 的逗号分隔列表。每个所选发布者保留自己的配置命名空间。

```properties
conductor.workflow-status-listener.type=composite
conductor.workflow-status-listener.composite.types=kafka,workflow_publisher,queue_publisher

conductor.workflow-status-listener.kafka.producer[bootstrap.servers]=kafka:29092
conductor.workflow-status-listener.kafka.default-topic=workflow-events
conductor.status-notifier.notification.url=https://example.internal/workflow-events
conductor.workflow-status-listener.queue-publisher.successQueue=_callbackSuccessQueue
conductor.workflow-status-listener.queue-publisher.failureQueue=_callbackFailureQueue
```

复合监听器独立创建每个已配置的发布者。某个所选发布者中的配置错误会阻止它的创建，因此在部署前请验证每个所选发布者的必需属性。

## 相关参考

- [工作流定义](../../documentation/configuration/workflowdef/index.md#workflow-status-listener) 记载了工作流级别的选择加入标志。
- [事件编排](event-bus.md) 记载了入站 broker 事件、事件处理器和提供商配置。
