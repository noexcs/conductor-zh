---
description: "使用 Saga 模式（含补偿流程）、重试策略、任务级错误处理、超时策略和工作流状态监听器通知来处理 Conductor 中的工作流错误。"
---

# 处理工作流错误

在生产微服务架构中，故障不可避免。Conductor 提供多层错误处理机制，帮助你构建具有弹性、可自愈的工作流：

* **Saga 模式** — 工作流失败时运行补偿流程，撤销已完成的步骤。
* **重试策略** — 使用可配置的回退自动重试失败任务。
* **任务级错误处理** — 将任务标记为可选、在终止性错误时立即失败，或为每个任务设置超时。
* **超时策略** — 控制任务或工作流超出时限时的行为。
* **工作流状态监听器** — 在工作流完成或失败时向外部系统发送通知。

## Saga 模式：失败时补偿

Saga 模式是管理跨微服务分布式事务的一种成熟方法。Saga 不是跨越多个服务的单一原子事务，而是将工作拆分为一系列本地事务。每个步骤都有相应的**补偿操作**来撤销其影响。当序列中任何步骤失败时，之前完成的步骤会执行其补偿操作，按相反顺序回滚。

在无法实用两阶段提交的微服务架构中，该模式至关重要。由于每个服务都拥有自己的数据，无法依赖传统数据库事务来维护跨服务一致性。Saga 模式以显式回滚逻辑提供最终一致性，使故障可预测且可恢复。

### 配置失败工作流

在主工作流定义中添加 `failureWorkflow` 参数，可配置工作流在失败时自动运行。
此外，还可以使用 `failureWorkflowVersion` 参数指定其_版本_。

```json
"failureWorkflow": "<Name of your compensation flow>",
"failureWorkflowVersion": 2,
```

如果主工作流失败，Conductor 会触发该失败工作流。默认情况下，以下参数会作为输入传递给失败工作流：

* **`reason`** — 工作流失败的原因。
* **`workflowId`** — 失败工作流的执行 ID。
* **`failureStatus`** — 失败工作流的状态。
* **`failureTaskId`** — 工作流中失败任务的执行 ID。
* **`failedWorkflow`** — 失败工作流的完整工作流执行 JSON。

可以使用这些参数在失败工作流中实现补偿操作，例如通知告警、资源清理或撤销已完成的事务。

### 示例：失败时发送 Slack 通知

下面是一个在主工作流失败时发送 Slack 消息的失败工作流。它会发布 `reason` 和 `workflowId`，以便团队调试该失败：

```json
{
  "name": "shipping_failure",
  "description": "Notification workflow for shipping workflow failures",
  "version": 1,
  "tasks": [
    {
      "name": "slack_message",
      "taskReferenceName": "send_slack_message",
      "inputParameters": {
        "http_request": {
          "headers": {
            "Content-type": "application/json"
          },
          "uri": "https://hooks.slack.com/services/<_unique_Slack_generated_key_>",
          "method": "POST",
          "body": {
            "text": "workflow: ${workflow.input.workflowId} failed. ${workflow.input.reason}"
          },
          "connectionTimeOut": 5000,
          "readTimeOut": 5000
        }
      },
      "type": "HTTP",
      "retryCount": 3
    }
  ],
  "restartable": true,
  "workflowStatusListenerEnabled": false,
  "ownerEmail": "conductor@example.com",
  "timeoutPolicy": "ALERT_ONLY"
}
```

### 示例：订单处理的 Saga 补偿

实际的 Saga 实现包含一个主工作流和一个补偿工作流：主工作流通过多个服务处理订单，若任何步骤失败，补偿工作流会逐一撤销每个已完成的步骤。

**主工作流** — `order_processing` 通过三个阶段处理客户订单：扣款、预留库存、安排发货。

```json
{
  "name": "order_processing",
  "description": "Process a customer order through payment, inventory, and shipping",
  "version": 1,
  "failureWorkflow": "order_compensation",
  "tasks": [
    {
      "name": "charge_payment",
      "taskReferenceName": "charge_payment_ref",
      "inputParameters": {
        "orderId": "${workflow.input.orderId}",
        "customerId": "${workflow.input.customerId}",
        "amount": "${workflow.input.totalAmount}"
      },
      "type": "SIMPLE",
      "retryCount": 2,
      "retryLogic": "EXPONENTIAL_BACKOFF",
      "retryDelaySeconds": 5
    },
    {
      "name": "reserve_inventory",
      "taskReferenceName": "reserve_inventory_ref",
      "inputParameters": {
        "orderId": "${workflow.input.orderId}",
        "items": "${workflow.input.items}",
        "paymentTransactionId": "${charge_payment_ref.output.transactionId}"
      },
      "type": "SIMPLE",
      "retryCount": 2,
      "retryLogic": "FIXED",
      "retryDelaySeconds": 3
    },
    {
      "name": "arrange_shipping",
      "taskReferenceName": "arrange_shipping_ref",
      "inputParameters": {
        "orderId": "${workflow.input.orderId}",
        "shippingAddress": "${workflow.input.shippingAddress}",
        "items": "${workflow.input.items}",
        "inventoryReservationId": "${reserve_inventory_ref.output.reservationId}"
      },
      "type": "SIMPLE",
      "retryCount": 1,
      "retryLogic": "FIXED",
      "retryDelaySeconds": 10
    }
  ],
  "restartable": true,
  "workflowStatusListenerEnabled": true,
  "ownerEmail": "order-team@example.com",
  "timeoutPolicy": "TIME_OUT_WF",
  "timeoutSeconds": 600
}
```

**补偿工作流** — `order_compensation` 以相反顺序撤销每个已完成的步骤：取消发货、恢复库存、退款。

```json
{
  "name": "order_compensation",
  "description": "Undo completed order steps when order_processing fails",
  "version": 1,
  "tasks": [
    {
      "name": "cancel_shipment",
      "taskReferenceName": "cancel_shipment_ref",
      "inputParameters": {
        "orderId": "${workflow.input.failedWorkflow.input.orderId}",
        "shipmentId": "${workflow.input.failedWorkflow.tasks[arrange_shipping_ref].output.shipmentId}"
      },
      "type": "SIMPLE",
      "optional": true,
      "retryCount": 3,
      "retryLogic": "FIXED",
      "retryDelaySeconds": 5
    },
    {
      "name": "restore_inventory",
      "taskReferenceName": "restore_inventory_ref",
      "inputParameters": {
        "orderId": "${workflow.input.failedWorkflow.input.orderId}",
        "reservationId": "${workflow.input.failedWorkflow.tasks[reserve_inventory_ref].output.reservationId}"
      },
      "type": "SIMPLE",
      "optional": true,
      "retryCount": 3,
      "retryLogic": "FIXED",
      "retryDelaySeconds": 5
    },
    {
      "name": "refund_payment",
      "taskReferenceName": "refund_payment_ref",
      "inputParameters": {
        "orderId": "${workflow.input.failedWorkflow.input.orderId}",
        "transactionId": "${workflow.input.failedWorkflow.tasks[charge_payment_ref].output.transactionId}",
        "amount": "${workflow.input.failedWorkflow.input.totalAmount}"
      },
      "type": "SIMPLE",
      "retryCount": 5,
      "retryLogic": "EXPONENTIAL_BACKOFF",
      "retryDelaySeconds": 10
    }
  ],
  "restartable": true,
  "workflowStatusListenerEnabled": false,
  "ownerEmail": "order-team@example.com",
  "timeoutPolicy": "ALERT_ONLY",
  "timeoutSeconds": 1200
}
```

注意，对于在失败发生前可能尚未完成的步骤，其补偿任务被标记为 `optional: true`。退款任务使用激进的指数回退重试，因为客户拿回钱至关重要。

## 重试策略

任务失败时，Conductor 可以根据任务定义上配置的重试逻辑自动重试。使用三个参数控制重试行为：

* **`retryCount`** — 最大重试次数。
* **`retryLogic`** — 重试之间的回退策略。
* **`retryDelaySeconds`** — 重试之间的基础延迟，单位为秒。

### FIXED

以恒定间隔重试。每次重试等待相同的时间。

```json
{
  "retryCount": 3,
  "retryLogic": "FIXED",
  "retryDelaySeconds": 5
}
```

最多重试 3 次，每次尝试之间恰好等待 5 秒。

### EXPONENTIAL_BACKOFF

每次重试等待的时间按指数递增，长于上一次。延迟按 `retryDelaySeconds * 2^(attemptNumber)` 计算。这降低了可能正承受压力的下游服务的负载。

```json
{
  "retryCount": 4,
  "retryLogic": "EXPONENTIAL_BACKOFF",
  "retryDelaySeconds": 2
}
```

最多重试 4 次，延迟约为 2、4、8、16 秒。

### LINEAR_BACKOFF

每次重试按固定量递增等待时间。延迟按 `retryDelaySeconds * attemptNumber` 计算。相比指数回退，它的爬坡更平缓。

```json
{
  "retryCount": 4,
  "retryLogic": "LINEAR_BACKOFF",
  "retryDelaySeconds": 5
}
```

最多重试 4 次，延迟约为 5、10、15、20 秒。

### 选择重试策略

| 策略 | 延迟模式 | 最适合 |
|---|---|---|
| `FIXED` | 恒定（例如 5s、5s、5s） | 可预测的瞬时故障，如短暂网络抖动或短时的锁竞争。 |
| `EXPONENTIAL_BACKOFF` | 倍增（例如 2s、4s、8s、16s） | 限流 API、过载服务，或任何希望降低对苦苦支撑的依赖施加压力的场景。 |
| `LINEAR_BACKOFF` | 递增（例如 5s、10s、15s、20s） | 温和的恢复场景：需要随时间逐渐加长等待，但指数增长过于激进。 |

## 任务级错误处理

除重试外，Conductor 还提供若干任务级控制项，用于管理运行中工作流内的故障。

### 可选任务

将任务的 `optional` 设为 `true`，表示即使该任务在耗尽所有重试后仍失败，Conductor 也会继续工作流。工作流会继续到下一个任务，而不是整体失败。

```json
{
  "name": "send_analytics_event",
  "taskReferenceName": "send_analytics_ref",
  "type": "SIMPLE",
  "optional": true,
  "retryCount": 2,
  "retryLogic": "FIXED",
  "retryDelaySeconds": 3
}
```

对于日志、分析、通知等失败不应阻断主要业务逻辑的非关键副作用，使用可选任务。

### 用终止性错误立即失败

当工作者遇到无论重试多少次都无法修复的错误（如无效输入数据或违反业务规则）时，应返回 `FAILED_WITH_TERMINAL_ERROR` 状态。这会让 Conductor 跳过所有剩余重试并立即使任务失败。

工作者通过在任务结果中将任务状态设为 `FAILED_WITH_TERMINAL_ERROR` 来发出该信号。当失败是确定性的时，这避免了在重试上浪费时间。例如，如果付款因余额不足被拒，重试同一笔扣款永远不会成功。

### 按任务配置超时

可以为单个任务设置超时，防止其无限期阻塞工作流：

```json
{
  "name": "call_external_api",
  "taskReferenceName": "call_api_ref",
  "type": "SIMPLE",
  "timeoutSeconds": 120,
  "responseTimeoutSeconds": 60,
  "timeoutPolicy": "RETRY"
}
```

* **`timeoutSeconds`** — 任务的最大总时间，包括所有重试。
* **`responseTimeoutSeconds`** — 等待工作者领取并响应任务的最大时间。若工作者未在该时间窗口内更新任务，Conductor 将其标记为超时。

## 超时策略

超时策略决定任务超过其 `timeoutSeconds` 或 `responseTimeoutSeconds` 限制时 Conductor 的行为。

### RETRY

将任务重新入队再试一次。该重试计入任务的 `retryCount`。

```json
{
  "timeoutPolicy": "RETRY",
  "timeoutSeconds": 60,
  "retryCount": 3
}
```

### TIME_OUT_WF

任务超时时立即使整个工作流失败。用于超时即表示存在使继续工作流变得无意义的严重问题的任务。

```json
{
  "timeoutPolicy": "TIME_OUT_WF",
  "timeoutSeconds": 300
}
```

### ALERT_ONLY

记录告警，但允许任务继续运行。任务不会被终止或重试。对于希望在长时间运行的任务上获得慢速执行可见性又不中断工作的场景，此策略很有用。

```json
{
  "timeoutPolicy": "ALERT_ONLY",
  "timeoutSeconds": 600
}
```

### 选择超时策略

| 策略 | 超时时行为 | 最适合 |
|---|---|---|
| `RETRY` | 重试该任务（计入 `retryCount`） | 可能因网络超时或无响应工作者等瞬时问题而挂起的任务。 |
| `TIME_OUT_WF` | 使整个工作流失败 | 超时即意味着工作流无法产生有效结果的关键任务。 |
| `ALERT_ONLY` | 记录告警，任务继续运行 | 只需监控而不强制干预的长时间运行或尽力而为的任务。 |

## 实现工作流状态监听器

使用工作流状态监听器，可以在失败时向外部系统发送通知，或向 Conductor 的内部队列发送事件。以下是使用工作流状态监听器的高层概览：

1. 在主工作流定义中将 `workflowStatusListenerEnabled` 参数设为 true：
    ```json
    "workflowStatusListenerEnabled": true,
    ```
2. 实现 [WorkflowStatusListener 接口](https://github.com/conductor-oss/conductor/blob/1be02a711dc20682718c6111c09d2b02ce7edde2/core/src/main/java/com/netflix/conductor/core/listener/WorkflowStatusListener.java#L20)，以便在工作流失败时接入自定义通知或事件系统。
