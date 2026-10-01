---
description: 通过激活的事件处理器路由 broker 消息，以启动工作流，或精确地完成或使已识别的任务失败。
---

# 消费与路由事件

<section class="concept-hero concept-hero--event-bus" aria-label="消费与路由事件">
  <div class="concept-hero__content">
    <p>一个<strong>事件处理器</strong>是已注册的规则，它从 broker 消费消息并将其转化为工作流动作。当消息到达处理器所监视的队列时，处理器根据载荷评估其条件，可以启动一个新工作流，或完成或使某个特定任务失败。处理器就是外部系统在不直接调用 Conductor API 的情况下驱动工作流的方式。</p>
  </div>
  <svg class="concept-hero__graphic event-hero__graphic" viewBox="0 0 440 190" role="img" aria-labelledby="consume-svg-title consume-svg-desc" xmlns="http://www.w3.org/2000/svg">
    <title id="consume-svg-title">事件处理器路由流程</title>
    <desc id="consume-svg-desc">broker 消息到达事件处理器。其条件和评估器导向工作流启动，或精确的任务完成或失败。</desc>
    <defs><marker id="consume-arrow" markerWidth="8" markerHeight="8" refX="7" refY="4" orient="auto"><path d="M0,0 L8,4 L0,8 z" fill="currentColor"/></marker></defs>
    <rect x="14" y="68" width="101" height="54" rx="10" class="concept-hero__node event-hero__node--broker"/><text x="64" y="91" text-anchor="middle" class="concept-hero__label">Broker 事件</text><text x="64" y="108" text-anchor="middle" class="concept-hero__detail">消息 + ID</text>
    <path d="M115 95 H153" class="concept-hero__line" marker-end="url(#consume-arrow)"/>
    <rect x="161" y="58" width="119" height="74" rx="10" class="concept-hero__node concept-hero__node--accent"/><text x="220" y="83" text-anchor="middle" class="concept-hero__label">事件处理器</text><text x="220" y="100" text-anchor="middle" class="concept-hero__detail">条件 + 评估器</text><text x="220" y="117" text-anchor="middle" class="concept-hero__detail">匹配的动作</text>
    <path d="M280 81 H317" class="concept-hero__line" marker-end="url(#consume-arrow)"/>
    <path d="M280 109 H300 V146 H317" class="concept-hero__line" marker-end="url(#consume-arrow)"/>
    <rect x="325" y="55" width="101" height="44" rx="10" class="concept-hero__node event-hero__node--action"/><text x="375" y="82" text-anchor="middle" class="concept-hero__label">启动工作流</text>
    <rect x="325" y="124" width="101" height="44" rx="10" class="concept-hero__node event-hero__node--action"/><text x="375" y="145" text-anchor="middle" class="concept-hero__label">精确任务</text><text x="375" y="160" text-anchor="middle" class="concept-hero__detail">完成或失败</text>
  </svg>
</section>

## 注册处理器

使用[事件处理器 API](../../documentation/api/eventhandlers.md) 创建并激活处理器。其 `event` 为 `provider:<提供商特定队列 URI>`；运行时解析在第一个冒号处分割。该提供商必须在服务器上启用。

在 Orkes 上，先配置托管的 broker 集成，然后在事件处理器流程中使用该已配置的集成。下面的 OSS API 示例使用的是 OSS 提供商键和已启用的服务器模块；它不是集成配置示例。

```json
{
  "name": "start_fulfillment_on_order_ready",
  "event": "conductor:publish_order_event:order-status",
  "condition": "$.status == 'READY'",
  "actions": [
    {
      "action": "start_workflow",
      "start_workflow": {
        "name": "fulfill_order",
        "version": 1,
        "correlationId": "${orderId}",
        "input": {
          "orderId": "${orderId}",
          "sourceEventId": "${workflowInstanceId}"
        }
      }
    }
  ],
  "active": true
}
```

## 匹配载荷本身，而不是外层包装

条件和占位符直接以交付的载荷为根。例如，条件中使用 `$.status == 'READY'`，动作中使用 `${orderId}`。缺失的条件视为真；`active` 默认为 `false`。

如果 `evaluatorType` 指定了已注册的评估器，Conductor 会使用它。否则使用默认的脚本评估器。仅当事件内的字段是有意为之的 JSON 字符串、且必须在表达式解析前展开时，才在动作上设置 `expandInlineJSON: true`。

## 选择动作

| 动作 | OSS Conductor | Orkes | 行为 |
|---|:---:|:---:|---|
| `start_workflow` | 是 | 是 | 启动指定名称的工作流，并在其输入中包含 Conductor 事件元数据。 |
| `complete_task` | 是 | 是 | 完成一个已识别的任务。 |
| `fail_task` | 是 | 是 | 使一个已识别的任务失败，并可设置 `reasonForIncompletion`。 |
| `terminate_workflow` | 否 | 是 | 终止目标工作流。 |
| `update_workflow_variables` | 否 | 是 | 更新目标工作流上的变量。 |

任务动作需要精确的目标：提供 `taskId`，或同时提供 `workflowId` 和 `taskRefName`。仅有业务关联键无法把一个 OSS 处理器动作解析到等待中的任务。

## 完成或使目标任务失败

当事件本身提供了任务身份时，使用任务动作。处理器从 broker 载荷中解析占位符。

当审批事件到达时完成任务：

```json
{
  "name": "complete_payment_wait",
  "event": "kafka:payment-events",
  "condition": "$.status == 'APPROVED'",
  "actions": [
    {
      "action": "complete_task",
      "complete_task": {
        "workflowId": "${workflowId}",
        "taskRefName": "wait_for_payment",
        "output": {
          "paymentId": "${paymentId}",
          "approved": true
        }
      }
    }
  ],
  "active": true
}
```

当事件被拒绝且应使任务失败时，注册一个单独的处理器：

```json
{
  "name": "fail_payment_wait",
  "event": "kafka:payment-events",
  "condition": "$.status == 'REJECTED'",
  "actions": [
    {
      "action": "fail_task",
      "fail_task": {
        "taskId": "${rejectionTaskId}",
        "reasonForIncompletion": "${reason}",
        "output": {
          "providerStatus": "${status}"
        }
      }
    }
  ],
  "active": true
}
```

## 投递与幂等性

动作并发执行且不是原子的。Conductor 用 broker 消息 ID 和动作索引记录每个动作；稳定的消息 ID 使得在事件执行存储之后可以进行持久化的去重检测。即便如此，仍要让工作流启动、任务更新以及任何外部副作用保持幂等。当条件为假时，Conductor 记录一次被跳过的事件执行，且不运行任何动作。

## 后续步骤

<div class="event-next-steps">
  <a href="publish-events.html">发布事件 →</a>
  <a href="incoming-webhooks.html">接收 HTTP 回调 →</a>
  <a href="../../documentation/configuration/eventhandlers.html">事件处理器参考 →</a>
  <a href="../cookbook/sending-signals.html">向已知等待中的工作流发送信号 →</a>
</div>
