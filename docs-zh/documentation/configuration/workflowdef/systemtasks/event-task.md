---
description: "EVENT 系统任务的输入、载荷、sink 扩展以及异步完成行为。"
---

# 使用 Event 任务发布事件

`EVENT` 通过已注册的事件队列提供方发布 JSON 消息。它是通用的发布任务：当消息契约需要 Kafka 特有的 key、headers、序列化器或生产者控制时，请使用 [`KAFKA_PUBLISH`](kafka-publish-task.md)。

## 任务参数

| 参数 | 必填 | 行为 |
|---|---|---|
| `sink` | 是 | `provider:<provider 特定目标>`；表达式在运行时解析 |
| `inputParameters` | 否 | 用户载荷字段 |
| `asyncComplete` | 否 | 默认为 `false`；为 true 时任务在发布后保持 `IN_PROGRESS` |

在 OSS 中，已注册的提供方标识符有 `conductor`、`kafka`、`sqs`、`nats`、`jsm`、`nats_stream`、`amqp_queue` 和 `amqp_exchange`，具体取决于相应的服务器模块是否启用。提供方拥有第一个冒号之后的目标语法；例如，它可能是 Kafka 主题、SQS 队列 URL、NATS subject 或 AMQP 队列/交换器。

## Conductor sink 扩展

- `conductor` 变为 `conductor:<workflowName>:<taskReferenceName>`。
- `conductor:<suffix>` 变为 `conductor:<workflowName>:<suffix>`。

事件处理器必须监听扩展后的名称。

## 发布的载荷与输出

任务以其解析后的输入参数开始，并添加工作流元数据：

| 字段 | 值 |
|---|---|
| `workflowInstanceId` | 父工作流执行 ID |
| `workflowType` | 父工作流名称 |
| `workflowVersion` | 父版本号 |
| `correlationId` | 父关联 ID |
| `taskToDomain` | 父域映射 |

任务输出还包含 `event_produced`，即扩展后的 sink。发布的消息是去除 `event_produced` 后的任务输出。Event 任务使用其任务 ID 作为 broker 消息标识，因此消费者可以使用该稳定值进行去重检测。

## 完成行为

当 `asyncComplete: false` 时，发布成功即完成任务。当 `asyncComplete: true` 时，发布成功但任务保持 `IN_PROGRESS`；必须由外部任务更新或事件处理器的 `complete_task`/`fail_task` 动作来解决。

## 示例

```json
{
  "name": "publish_order_status",
  "taskReferenceName": "publish_order_status",
  "type": "EVENT",
  "sink": "conductor:order-status",
  "inputParameters": {
    "orderId": "${workflow.input.orderId}",
    "status": "READY"
  },
  "asyncComplete": false
}
```

要查看实际的首次使用步骤，请参阅[发布事件](../../../../devguide/how-tos/publish-events.md)。关于提供方矩阵、路由、webhook、信号和投递可观测性，请参阅[事件驱动编排](../../../../devguide/how-tos/event-bus.md)。

<a id="configuration-json"></a>
<a id="conductor-sink-configuration"></a>
<a id="output"></a>
<a id="examples"></a>
