---
description: 用 EVENT 把已解析的工作流数据发布到已启用的事件 sink，或仅在需要 Kafka 专属控制时使用 KAFKA_PUBLISH。
---

# 发布事件

<section class="concept-hero concept-hero--event-bus" aria-label="发布事件">
  <div class="concept-hero__content">
    <p>工作流可以用<strong><code>EVENT</code> 任务</strong>向外部世界发送消息。该任务接收已解析的工作流数据，添加持久化元数据，并把消息发布到已配置的提供商（如 Kafka 或 SQS）。发布是执行中的一个普通步骤，因此会像其他任务一样被记录和重试。仅在需要 Kafka 专属的生产者控制时使用 <code>KAFKA_PUBLISH</code>。</p>
  </div>
  <svg class="concept-hero__graphic event-hero__graphic" viewBox="0 0 440 190" role="img" aria-labelledby="publish-svg-title publish-svg-desc" xmlns="http://www.w3.org/2000/svg">
    <title id="publish-svg-title">工作流事件发布流程</title>
    <desc id="publish-svg-desc">工作流把已解析的输入发送给 Event 任务，该任务会发布到为 broker 和消费者配置的 sink。Kafka Publish 是另一个 Kafka 专属选项。</desc>
    <defs><marker id="publish-arrow" markerWidth="8" markerHeight="8" refX="7" refY="4" orient="auto"><path d="M0,0 L8,4 L0,8 z" fill="currentColor"/></marker></defs>
    <rect x="14" y="59" width="92" height="54" rx="10" class="concept-hero__node"/><text x="60" y="82" text-anchor="middle" class="concept-hero__label">工作流</text><text x="60" y="99" text-anchor="middle" class="concept-hero__detail">已解析输入</text>
    <path d="M106 86 H145" class="concept-hero__line" marker-end="url(#publish-arrow)"/>
    <rect x="153" y="59" width="88" height="54" rx="10" class="concept-hero__node concept-hero__node--accent"/><text x="197" y="82" text-anchor="middle" class="concept-hero__label">EVENT</text><text x="197" y="99" text-anchor="middle" class="concept-hero__detail">元数据</text>
    <path d="M241 86 H279" class="concept-hero__line" marker-end="url(#publish-arrow)"/>
    <rect x="287" y="59" width="139" height="54" rx="10" class="concept-hero__node event-hero__node--broker"/><text x="356" y="82" text-anchor="middle" class="concept-hero__label">已配置的 sink</text><text x="356" y="99" text-anchor="middle" class="concept-hero__detail">broker 或消费者</text>
    <path d="M197 113 V143 H317" class="concept-hero__line event-hero__line--dashed" marker-end="url(#publish-arrow)"/>
    <text x="194" y="164" text-anchor="middle" class="concept-hero__detail">KAFKA_PUBLISH：Kafka 专属分支</text>
  </svg>
</section>

## 选择任务

| 用途 | 适用场景 | 它能提供什么 |
|---|---|---|
| `EVENT` | 你想向已启用的事件队列提供商发送提供商中立的消息。 | 通用的 sink 模型、工作流元数据、稳定的消息标识，以及与事件处理器的兼容性。 |
| `KAFKA_PUBLISH` | 你的契约需要 Kafka 专属的 key、header、序列化器或生产者控制。 | 带 Kafka 专属配置的直接 Kafka 主题发布。 |

不要仅仅因为目标恰好是 Kafka 就使用 `KAFKA_PUBLISH`。除非确实需要那些 Kafka 专属控制，否则优先使用 `EVENT`。

## 命名目标

`EVENT` sink 的格式是 `provider:<提供商特定目标>`。在 OSS 中，已启用的提供商键包括 `conductor`、`kafka`、`sqs`、`nats`、`jsm`、`nats_stream`、`amqp_queue` 和 `amqp_exchange`。

`conductor` 提供商会把简写 sink 展开为按工作流命名空间的形式：

| 定义中的 sink | 工作流 `order_workflow` 展开后的 sink |
|---|---|
| `conductor` | `conductor:order_workflow:<taskReferenceName>` |
| `conductor:order-status` | `conductor:order_workflow:order-status` |

事件处理器必须订阅展开后的名称。Kafka 主题、SQS 队列 URL、NATS subject 和 AMQP 目标保留其提供商要求的语法。

在 Orkes 上，选择为租户配置的托管 broker 集成，并使用其带集成限定的 sink 命名。该配置与上述 OSS 提供商键不同：不要把 OSS 提供商前缀复制到 Orkes 集成名中，也不要假设 Orkes 集成名可以移植到 OSS。

## 发布的内容

Conductor 解析 `inputParameters`，然后向发布的 JSON 添加这些字段：

| 字段 | 值 |
|---|---|
| `workflowInstanceId` | 父工作流执行 ID |
| `workflowType` / `workflowVersion` | 父工作流名称和版本 |
| `correlationId` | 父关联 ID |
| `taskToDomain` | 父任务-域映射 |

任务输出还包含 `event_produced` 和展开后的 sink，但该字段不会作为 broker 消息的一部分发送。Event 任务 ID 就是 broker 消息的身份标识；消费者可以把它用作持久化的去重键。

## 发布一个订单状态事件

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

当 `asyncComplete: false` 时，broker 发布成功即完成任务。当 `asyncComplete: true` 时，发布成功但任务保持 `IN_PROGRESS`，直到外部任务更新或事件处理器的 `complete_task` 或 `fail_task` 动作将其解决。

## 生产环境建议

- **投递：** 把 broker 投递当作至少一次（at-least-once）处理。消费者动作和任何副作用必须幂等。
- **可观测性：** 监控 `event_queue_depth` 和 `event_queue_messages_*` 计数器，然后检查下游工作流或任务的结果。
- **身份标识：** 保留 broker 消息 ID 并使用 Event 任务 ID 做去重；不要为重试编造新的随机 key。

## 后续步骤

<div class="event-next-steps">
  <a href="consume-route-events.html">路由已发布的事件 →</a>
  <a href="incoming-webhooks.html">改为接收 HTTP 回调 →</a>
  <a href="../../documentation/configuration/workflowdef/systemtasks/event-task.html">EVENT 任务参考 →</a>
  <a href="../../documentation/configuration/workflowdef/systemtasks/kafka-publish-task.html">KAFKA_PUBLISH 参考 →</a>
</div>
