---
description: "在 Conductor 中配置 Wait 任务，将工作流执行暂停固定时长或直到特定时间戳。支持持久化代码执行模式。"
---

# Wait 任务
```json
"type" : "WAIT"
```

Wait 任务（`WAIT`）用于将工作流暂停到某个时长或时间戳。它是一个空操作任务，会保持 IN_PROGRESS 状态，直到配置的时长过去，然后被标记为 COMPLETED。


## 任务参数

在 Wait 任务配置中，在 `inputParameters` 内使用这些参数。你可以在 `inputParameters` 中使用 `duration` 或 `until` 之一来配置 Wait 任务。

| 参数          | 类型                | 说明                                       | 必填 / 可选  |
| ------------------ | ------------------- | ------------------------------------------------- | -------------------- |
| duration | String | 格式为 `x days y hours z minutes aa seconds` 的等待时长。该字段接受的单位为：<ul><li>**days**，或 **d** 表示天</li> <li>**hours**、**hrs** 或 **h** 表示小时</li> <li>**minutes**、**mins** 或 **m** 表示分钟</li> <li>**seconds**、**secs** 或 **s** 表示秒</li></ul>   | 时长等待类型必填。 |
| until    | String | 等待至的目标日期时间和时区，格式为以下之一：<ul><li>yyyy-MM-dd HH:mm z</li> <li>yyyy-MM-dd HH:mm</li> <li>yyyy-MM-dd</li></ul> <br/> 例如，2024-04-30 15:20 GMT+04:00。 | until 等待类型必填。 |

## JSON 配置

以下是 Wait 任务的任务配置。

### 使用 `duration`

```json
{
	"name": "wait",
    "taskReferenceName": "wait_ref",
	"inputParameters": {
		"duration": "10m20s"
	},
	"type": "WAIT"
}
```

### 使用 `until`

```json
{
	"name": "wait",
    "taskReferenceName": "wait_ref",
	"inputParameters": {
		"until": "2022-12-31 11:59"
	},
	"type": "WAIT"
}
```

## 示例

### 等待固定时长

继续执行前等待 30 秒：

```json
{
  "name": "wait_30s",
  "taskReferenceName": "wait_30s_ref",
  "type": "WAIT",
  "inputParameters": {
    "duration": "30 seconds"
  }
}
```

等待 2 小时 30 分钟：

```json
{
  "name": "wait_2h30m",
  "taskReferenceName": "wait_2h30m_ref",
  "type": "WAIT",
  "inputParameters": {
    "duration": "2 hours 30 minutes"
  }
}
```

### 等待到特定日期/时间

等待到特定时间戳：

```json
{
  "name": "wait_until_deadline",
  "taskReferenceName": "wait_deadline_ref",
  "type": "WAIT",
  "inputParameters": {
    "until": "2025-06-15 09:00 GMT+00:00"
  }
}
```

等待到作为工作流输入提供的日期/时间：

```json
{
  "name": "wait_until_input_time",
  "taskReferenceName": "wait_input_ref",
  "type": "WAIT",
  "inputParameters": {
    "until": "${workflow.input.scheduledTime}"
  }
}
```

### 等待外部信号（无时长）

当未指定 `duration` 或 `until` 时，Wait 任务会无限期暂停，直到通过任务更新 API 或事件处理器在外部完成：

```json
{
  "name": "wait_for_signal",
  "taskReferenceName": "signal_ref",
  "type": "WAIT"
}
```

在外部完成任务：

```shell
curl -X POST 'http://localhost:8080/api/tasks/{workflowId}/signal_ref/COMPLETED/sync' \
  -H 'Content-Type: application/json' \
  -d '{"approvedBy": "admin"}'
```

## 覆盖 Wait 任务

可以使用任务更新 API（`POST api/tasks`）在配置的等待时长或时间戳到来之前，将 Wait 任务的状态设置为 COMPLETED。

如果工作流不需要特定的等待时长或时间戳，推荐直接使用 [Human](human-task.md) 任务，它会等待外部触发器。
