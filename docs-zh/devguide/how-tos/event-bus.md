---
description: 接收 broker 事件和 Webhook、发布工作流事件，或向已阻塞在 WAIT 上的工作流发送信号。
---

# 事件驱动编排

<section class="concept-hero concept-hero--event-bus" aria-labelledby="event-overview-title">
  <div class="concept-hero__content">
    <p>事件驱动编排把工作流与周围的消息连接起来。工作流可以向 broker 发布消息，传入的消息或 Webhook 可以启动或推进工作流，而信号可以恢复某个正在等待的特定执行。本节每个页面覆盖其中一个方向，下表会把你带到正确的页面。</p>
  </div>
  <svg class="concept-hero__graphic event-hero__graphic" viewBox="0 0 440 220" role="img" aria-labelledby="event-overview-svg-title event-overview-svg-desc" xmlns="http://www.w3.org/2000/svg">
    <title id="event-overview-svg-title">事件驱动编排路径</title>
    <desc id="event-overview-svg-desc">工作流向 broker 发布，事件处理器可将其路由到工作流或任务。Webhook 是经验证的 HTTP 入口，而信号直接推进被阻塞的等待任务。</desc>
    <defs><marker id="event-overview-arrow" markerWidth="8" markerHeight="8" refX="7" refY="4" orient="auto"><path d="M0,0 L8,4 L0,8 z" fill="currentColor"/></marker></defs>
    <rect x="14" y="16" width="103" height="48" rx="10" class="concept-hero__node concept-hero__node--accent"/><text x="66" y="37" text-anchor="middle" class="concept-hero__label">工作流</text><text x="66" y="53" text-anchor="middle" class="concept-hero__detail">EVENT</text>
    <path d="M117 40 H153" class="concept-hero__line" marker-end="url(#event-overview-arrow)"/>
    <rect x="161" y="16" width="105" height="48" rx="10" class="concept-hero__node event-hero__node--broker"/><text x="214" y="37" text-anchor="middle" class="concept-hero__label">Broker</text><text x="214" y="53" text-anchor="middle" class="concept-hero__detail">主题或队列</text>
    <path d="M266 40 H298" class="concept-hero__line" marker-end="url(#event-overview-arrow)"/>
    <rect x="306" y="16" width="120" height="48" rx="10" class="concept-hero__node event-hero__node--action"/><text x="366" y="37" text-anchor="middle" class="concept-hero__label">处理器</text><text x="366" y="53" text-anchor="middle" class="concept-hero__detail">启动或更新</text>
    <rect x="14" y="103" width="112" height="48" rx="10" class="concept-hero__node event-hero__node--broker"/><text x="70" y="124" text-anchor="middle" class="concept-hero__label">Webhook</text><text x="70" y="140" text-anchor="middle" class="concept-hero__detail">经验证的 HTTP</text>
    <path d="M126 127 H174" class="concept-hero__line" marker-end="url(#event-overview-arrow)"/>
    <rect x="182" y="103" width="109" height="48" rx="10" class="concept-hero__node event-hero__node--action"/><text x="236" y="124" text-anchor="middle" class="concept-hero__label">持久化工作</text><text x="236" y="140" text-anchor="middle" class="concept-hero__detail">启动或恢复</text>
    <rect x="14" y="172" width="112" height="34" rx="10" class="concept-hero__node event-hero__node--broker"/><text x="70" y="194" text-anchor="middle" class="concept-hero__label">信号调用方</text>
    <path d="M126 189 H174" class="concept-hero__line" marker-end="url(#event-overview-arrow)"/>
    <rect x="182" y="172" width="109" height="34" rx="10" class="concept-hero__node concept-hero__node--accent"/><text x="236" y="194" text-anchor="middle" class="concept-hero__label">阻塞的 WAIT</text>
    <path d="M291 189 H341" class="concept-hero__line" marker-end="url(#event-overview-arrow)"/>
    <text x="383" y="194" text-anchor="middle" class="concept-hero__detail">继续</text>
  </svg>
</section>

| 需求 | 从这里开始 | 可用性 |
|---|---|---|
| 把工作流数据发布到队列或 broker | [发布事件](publish-events.md) | OSS 和 Orkes |
| 消费 broker 消息并启动或更新工作流工作 | [消费与路由事件](consume-route-events.md) | OSS 和 Orkes |
| 接收外部服务的 HTTP 回调 | [传入 Webhook](incoming-webhooks.md) | 仅 Orkes |
| 继续阻塞在 `WAIT` 上的工作流 | [向工作流发送信号](../cookbook/sending-signals.md) | OSS 和 Orkes |
| 执行状态变化时通知外部系统 | [工作流状态事件](workflow-status-events.md) | OSS 和 Orkes |

`EVENT` 发布消息；事件处理器消费并路由它们。Webhook 是 HTTP 入口，不是通用的事件处理器。信号改变的是已存在的工作流，不会创建新的执行。

## Broker 提供商矩阵

提供商支持取决于 Conductor 发行版和已启用的服务器集成。事件名中第一个冒号之后的目标是提供商特定的。

| 提供商 | OSS Conductor | Orkes |
|---|:---:|:---:|
| Conductor 内部队列 | 是 | — |
| Kafka | 是 | 是 |
| Amazon SQS | 是 | 是 |
| NATS | 是 | 是 |
| NATS JetStream | 是 | — |
| NATS Streaming | 是 | — |
| AMQP 队列 / 交换机 | 是 | 是（包括 RabbitMQ） |
| Azure Service Bus | — | 是 |
| Google Cloud Pub/Sub | — | 是 |
| IBM MQ | — | 是 |

## 运维整条路径

监控 broker 队列深度（`event_queue_depth`）、消息处理（`event_queue_messages_processed`、`event_queue_messages_handled` 和 `event_queue_messages_error`）以及处理器动作（`event_execution_success` 和 `event_execution_error`）。然后检查由此产生的工作流或任务：仅有 broker 的确认并不能证明下游动作达到了预期状态。

## 后续步骤

- **[发布事件](publish-events.md)** — 把工作流数据发送到 broker。
- **[消费与路由事件](consume-route-events.md)** — 从传入消息启动或推进工作流。
- **[传入 Webhook](incoming-webhooks.md)** — 接收经验证的 HTTP 回调。
- **[发送信号](../cookbook/sending-signals.md)** — 推进正在等待的执行。
- **[工作流状态事件](workflow-status-events.md)** — 在执行状态变化时通知外部系统。
