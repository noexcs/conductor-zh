---
description: 事件处理器模型、表达式作用域、支持的动作及 OSS 运行时语义。
---

# 使用事件处理器消费与路由事件

事件处理器消费一个来自提供方（provider）的事件，评估可选的条件，并派发一个或多个动作。通过 [Event Handlers API](../api/eventhandlers.md) 注册；只有处于激活状态的处理器才会被订阅并处理。

```json
--8<-- "docs/devguide/cookbook/examples/events/start-workflow-handler.json"
```

## 事件标识符

格式为 `provider:<provider-specific queue URI>`。运行时解析在第一个冒号处拆分。当其模块被启用时，有效的已注册提供方键为 `conductor`、`kafka`、`sqs`、`nats`、`jsm`、`nats_stream`、`amqp_queue` 和 `amqp_exchange`。

## 条件与负载表达式

- `active` 默认为 `false`。
- 缺省的条件视为 true。
- 条件针对负载根节点求值，例如 `$.status == 'READY'`。若 `evaluatorType` 标识了已注册的求值器，Conductor 将使用它；否则使用默认脚本求值器对该条件求值。
- 动作占位符同样从负载根节点解析，例如 `${orderId}`。
- `expandInlineJSON: true` 会在表达式解析之前展开字符串化的 JSON 字段。

## 动作能力矩阵

| 动作 | OSS Conductor | Orkes | 行为 |
|---|:---:|:---:|---|
| `start_workflow` | 是 | 是 | 启动指定名称的工作流，并向其输入中添加 Conductor 事件元数据 |
| `complete_task` | 是 | 是 | 完成指定任务 |
| `fail_task` | 是 | 是 | 使指定任务失败；可设置 `reasonForIncompletion` |
| `terminate_workflow` | 否 | 是 | 终止目标工作流 |
| `update_workflow_variables` | 否 | 是 | 更新目标工作流上的变量 |

对于 `complete_task` 和 `fail_task`，请指定 `taskId`，或同时指定 `workflowId` 与 `taskRefName`。这些是精确的任务定位机制；OSS 处理器不会把业务关联键解析为等待中的任务。`terminate_workflow` 与 `update_workflow_variables` 存在于共享模型中，但 OSS 的动作处理器并未实现它们。

## 并发与去重

动作并发执行，且不具备原子性。每个动作用"broker 消息 ID + 其动作索引"分别记录。稳定的 broker 消息 ID 可在事件执行记录存储之后支持持久化的重复检测，但下游工作流启动、任务更新以及外部副作用仍然需要幂等性保证。

对于求值为 false 的条件，Conductor 会记录一次被跳过的事件执行，并且不运行任何动作。关于实际首次使用的演练步骤，请参阅 [消费与路由事件](../../devguide/how-tos/consume-route-events.md)；本页可用作动作与表达式的参考。
