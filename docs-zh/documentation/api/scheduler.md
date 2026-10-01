---
description: OSS Conductor 调度器的精确 REST 端点、请求字段、默认值和响应。
---

# 调度器 API

调度器控制器挂载在 `/api/scheduler`。仅当 `conductor.scheduler.enabled=true` 时它才存在。除特别说明外，以下所有端点在成功时都返回 `200 OK`。

## 调度模型

| 字段 | 类型 | 必填 | 运行时默认值或行为 |
|---|---|---|---|
| `name` | string | 是 | 用于创建或更新的唯一键 |
| `cronExpression` | string | 需要一种 cron 形式 | 遗留的单一表达式 |
| `zoneId` | string | 否 | `UTC` |
| `cronSchedules` | array | 需要一种 cron 形式 | 非空数组优先于 `cronExpression`/`zoneId`；条目的 `zoneId` 默认为 `UTC` |
| `startWorkflowRequest` | object | 是 | 标准的工作流启动请求 |
| `runCatchupScheduleInstances` | boolean | 否 | `false` |
| `paused` | boolean | 否 | `false` |
| `pausedReason` | string | 否 | 由暂停操作设置 |
| `scheduleStartTime` | long | 否 | 纪元毫秒下界 |
| `scheduleEndTime` | long | 否 | 纪元毫秒上界 |
| `description` | string | 否 | 用户描述 |
| `createTime`、`updatedTime`、`createdBy`、`updatedBy`、`nextRunTime` | 服务器字段 | 否 | 由服务填充 |

`cronSchedules` 条目包含 `cronExpression` 和可选的 `zoneId`。`startWorkflowRequest.correlationId` 会被原样复制。调度器会向工作流输入中添加 `_startedByScheduler`、`_scheduledTime`、`_executedTime`、`_executionId` 和 `_schedulerCron`。

## 创建或更新

```http
POST /api/scheduler/schedules
Content-Type: application/json
```

请求体是一个调度对象。响应是已存储的调度，包括计算出的状态（如 `nextRunTime`）。

```bash
curl -sS -X POST 'http://localhost:8080/api/scheduler/schedules' \
  -H 'Content-Type: application/json' \
  --data-binary @scheduler/examples/every-minute-schedule.json
```

## 列出与获取

```http
GET /api/scheduler/schedules?workflowName={workflowName}
GET /api/scheduler/schedules/{name}
```

`workflowName` 是可选的。列出返回数组；获取返回一个调度或服务端的未找到响应。

## 搜索调度

```http
GET /api/scheduler/schedules/search
```

| 查询 | 类型 | 默认值 |
|---|---|---|
| `workflowName` | string | 未设置 |
| `scheduleName` | string | 未设置 |
| `paused` | boolean | 未设置 |
| `freeText` | string | `*` |
| `start` | integer | `0` |
| `size` | integer | `100` |
| `sort` | 逗号分隔的字符串 | 空 |

返回 `SearchResult<WorkflowSchedule>`。

## 暂停与恢复

```http
PUT /api/scheduler/schedules/{name}/pause?reason={reason}
PUT /api/scheduler/schedules/{name}/resume
```

`reason` 是可选的。两个操作都返回空的 `200 OK` 响应。

## 批量暂停与恢复

```http
PUT /api/scheduler/bulk/pause
PUT /api/scheduler/bulk/resume
Content-Type: application/json
```

每个请求体都是调度名称的 JSON 数组。响应是一个 `BulkResponse`，包含成功的名称和按名称给出的错误。这些端点与调度器 API 的其余部分使用相同的调度器条件进行注册。

```json
["nightly-report", "hourly-cleanup"]
```

## 删除

```http
DELETE /api/scheduler/schedules/{name}
```

返回空的 `200 OK` 响应。

## 预览接下来的时间

```http
GET /api/scheduler/nextFewSchedules?cronExpression={cron}&scheduleStartTime={ms}&scheduleEndTime={ms}&limit={n}
```

`cronExpression` 必填。边界是可选的。`limit` 默认为 5，且实现会将结果上限设为 5。由于此端点不接受 `zoneId`，预览使用 `conductor.scheduler.schedulerTimeZone`，而不是请求或调度的时区。

## 搜索调度执行

```http
GET /api/scheduler/search/executions
```

| 查询 | 类型 | 默认值 |
|---|---|---|
| `query` | string | 未设置 |
| `freeText` | string | `*` |
| `start` | integer | `0` |
| `size` | integer | `100` |
| `sort` | 逗号分隔的字符串 | 空 |

返回 `SearchResult<WorkflowScheduleExecutionModel>`。执行记录包含调度器执行 ID、计划时间与执行时间、工作流名称/ID、状态，以及适用时的失败详情。

## 管理端点

```http
GET /api/scheduler/admin/requeue
GET /api/scheduler/admin/pause
GET /api/scheduler/admin/resume
```

这些端点操作调度器的内部机制，用于恢复/调试。它们不是按调度的暂停/恢复端点，应当进行访问控制。

## 不支持的操作

控制器没有立即运行（run-now）端点、手动补数据（backfill）端点、重叠策略字段，也没有关联模板展开。临时运行请直接启动工作流；并发/幂等策略请在工作流或下游系统中实现。
