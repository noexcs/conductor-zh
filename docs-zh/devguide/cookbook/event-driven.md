---
description: "可直接复制粘贴的 EVENT 和事件处理器配方，使用规范示例夹具。"
---

# 事件驱动配方

请先阅读[事件编排](../how-tos/event-bus.md)，了解操作支持、提供商配置、投递和幂等语义。

## 发布内部事件

```json
--8<-- "docs/devguide/cookbook/examples/events/publish-internal-event-workflow.json"
```

注册并运行工作流。其 `conductor:order-status` sink 会展开为 `conductor:publish_order_event:order-status`。

## 从事件启动工作流

先注册目标工作流：

```json
--8<-- "docs/devguide/cookbook/examples/events/fulfill-order-workflow.json"
```

```json
--8<-- "docs/devguide/cookbook/examples/events/start-workflow-handler.json"
```

```bash
curl -sS -X POST 'http://localhost:8080/api/event' \
  -H 'Content-Type: application/json' \
  --data-binary @docs/devguide/cookbook/examples/events/start-workflow-handler.json
```

payload 表达式直接以 Event 任务发布的 JSON 为根。

## 等待外部审批

工作流：

```json
--8<-- "docs/devguide/cookbook/examples/events/wait-for-approval-workflow.json"
```

处理器：

```json
--8<-- "docs/devguide/cookbook/examples/events/complete-wait-handler.json"
```

broker 的示例 payload：

```json
--8<-- "docs/devguide/cookbook/examples/events/approval-event.json"
```

将示例中的 `workflowId` 替换为等待中的工作流启动时返回的 ID。仅凭关联 ID 无法定位 WAIT 任务。

## 使用外部提供商

将 `event`/`sink` 改为已注册的提供商标识符及其特有的 URI，例如 `kafka:order-approvals`、`sqs:https://sqs.us-east-1.amazonaws.com/123/order-events`、`nats:orders.ready`、`jsm:orders.ready`、`nats_stream:orders.ready`、`amqp_queue:orders` 或 `amqp_exchange:orders`。启用指南中描述的对应模块和属性。
