---
description: OSS Conductor 事件处理器的 REST 端点与状态行为。
---

# 事件处理器 API

该控制器挂载在 `/api/event`。成功的变更操作返回空的 `200 OK` 响应。

## 端点

| 方法 | 路径 | 请求/响应 |
|---|---|---|
| `POST` | `/api/event` | 创建一个事件处理器对象；空响应 |
| `PUT` | `/api/event` | 替换/更新一个处理器对象；空响应 |
| `GET` | `/api/event` | 所有处理器的数组 |
| `DELETE` | `/api/event/{name}` | 按处理器名称移除；空响应 |
| `GET` | `/api/event/{event}?activeOnly=true` | 精确事件对应的处理器；`activeOnly` 默认为 `true` |

`{event}` 路径值可以包含提供方分隔符，需要时（由客户端/代理要求）必须进行 URL 编码。

## 创建示例

```bash
curl -sS -X POST 'http://localhost:8080/api/event' \
  -H 'Content-Type: application/json' \
  --data-binary @docs/devguide/cookbook/examples/events/start-workflow-handler.json
```

## 处理器字段

| 字段 | 必填 | 行为 |
|---|---|---|
| `name` | 是 | 非空、唯一的处理器名称 |
| `event` | 是 | `provider:<提供方特定的队列 URI>`；在第一个冒号处拆分 |
| `condition` | 否 | 针对负载根求值；省略视为 true |
| `actions` | 是 | 非空列表；动作并发执行 |
| `active` | 否 | 默认为 `false` |
| `evaluatorType` | 否 | 选择已注册的求值器；否则使用默认脚本求值器 |

共享模型声明了五个枚举值，但 OSS 的动作处理器只实现了 `start_workflow`、`complete_task` 和 `fail_task`。使用 `terminate_workflow` 或 `update_workflow_variables` 的请求可以反序列化，但在处理期间会因不受支持而失败。

## 任务定位

对于 `complete_task` 和 `fail_task`，请提供 `taskId`，或 `workflowId` 加上 `taskRefName`。`reasonForIncompletion` 仅对 `fail_task` 有意义。输出字段从事件负载根做表达式解析。

## 状态与投递行为

条件为 false 时会记录 `SKIPPED`。每个动作都有自己持久化的事件执行记录。去重依赖稳定的 broker 消息 ID 和持久化记录；动作并发执行，且不保证原子性。

数据模型参见 [事件处理器配置](../configuration/eventhandlers.md)，提供方配置与运维指引参见 [事件编排](../../devguide/how-tos/event-bus.md)。
