---
description: "Conductor 任务 API — 轮询、更新、搜索和管理任务。包括批量轮询、任务日志、队列管理和轮询数据。"
---

# 任务 API

任务响应是运行时对象。完整 schema 参见 [Task.json](../configuration/schemas.md#runtime-objects)，已注册的工作者任务配置参见 [TaskDef.json](../configuration/schemas.md#definition-objects)。

## 向被阻塞的任务发送信号

无需先解析任务 ID，即可向工作流中当前被阻塞的任务发送信号：

```http
POST /api/tasks/{workflowId}/{status}/signal
POST /api/tasks/{workflowId}/{status}/signal/sync
Content-Type: application/json
```

`status` 是 `TaskResult.Status`；请求体是任务输出映射。异步路由在发出信号后返回。同步路由会等待工作流的信号响应，并接受可选的 `returnStrategy`（默认 `TARGET_WORKFLOW`）和 `timeoutMillis`（默认 `5000`）。

任务 API 管理任务执行 — 轮询、更新、日志和队列管理。所有端点都使用基础路径 `/api/tasks`。

## 获取任务

```
GET /api/tasks/{taskId}
```

返回给定任务 ID 的任务详情。

```shell
curl 'http://localhost:8080/api/tasks/a1b2c3d4-5678-90ab-cdef-111111111111'
```

**响应** `200 OK`

```json
{
  "taskType": "my_task",
  "status": "COMPLETED",
  "referenceTaskName": "my_task_ref",
  "retryCount": 0,
  "seq": 1,
  "startTime": 1700000001000,
  "endTime": 1700000003000,
  "updateTime": 1700000003000,
  "pollCount": 1,
  "taskId": "a1b2c3d4-5678-90ab-cdef-111111111111",
  "workflowInstanceId": "3a5b8c2d-1234-5678-9abc-def012345678",
  "inputData": {"key": "value"},
  "outputData": {"result": "success"},
  "workerId": "worker-host-1"
}
```

**状态与失败字段**

| 字段 | 说明 |
|---|---|
| `status` | 当前任务状态。参见 [任务状态](../../devguide/architecture/tasklifecycle.md#task-statuses)。 |
| `reasonForIncompletion` | 任务未成功即停止的原因。健康时为空。由工作者、系统任务或引擎写入；截断为 500 个字符。参见 [理解 reasonForIncompletion](../../devguide/how-tos/Workflows/debugging-workflows.md#understanding-reasonforincompletion)。 |
| `retryCount` | 尝试次数，从 `0` 开始。 |
| `retried` | 当已为此任务调度后续尝试时为 `true`。 |
| `retriedTaskId` | 此任务所重试的较早尝试的 ID。 |
| `workerId` | 最后轮询或更新该任务的工作者实例。 |
| `pollCount` | 任务被轮询的次数。 |
| `callbackAfterSeconds` | `IN_PROGRESS` 更新之后，任务再次提供给工作者之前的延迟。 |
| `scheduledTime`、`startTime`、`endTime`、`updateTime` | 本次尝试的纪元毫秒数。 |

---

## 轮询与更新任务

这些端点供工作者轮询任务并更新其结果。通常由 [SDK](../clientsdks/index.md) 调用，而不是手动调用。

### 轮询任务

```
GET /api/tasks/poll/{taskType}?workerid=&domain=
```

轮询给定类型的单个任务。如果没有可用任务，返回 `204 No Content`。

| 参数 | 说明 | 必填 |
|---|---|---|
| `taskType` | 要轮询的任务类型 | 是 |
| `workerid` | 轮询的工作者的标识 | 否 |
| `domain` | 任务域。参见 [任务域](taskdomains.md)。 | 否 |

```shell
curl 'http://localhost:8080/api/tasks/poll/my_task?workerid=worker-1'
```

**响应** `200 OK` — 返回任务对象（与上文"获取任务"相同格式），如果没有任务排队则返回 `204 No Content`。

### 批量轮询

```
GET /api/tasks/poll/batch/{taskType}?count=1&timeout=100&workerid=&domain=
```

在单个请求中轮询多个任务。这是一个**长轮询** — 连接会一直等待，直到 `timeout` 或至少有一个任务可用。

| 参数 | 说明 | 默认值 |
|---|---|---|
| `taskType` | 要轮询的任务类型 | — |
| `count` | 返回的最大任务数 | `1` |
| `timeout` | 长轮询超时（毫秒） | `100` |
| `workerid` | 工作者标识 | — |
| `domain` | 任务域 | — |

```shell
# Poll for up to 5 tasks, wait up to 1 second
curl 'http://localhost:8080/api/tasks/poll/batch/my_task?count=5&timeout=1000&workerid=worker-1'
```

**响应** `200 OK` — 返回任务对象列表，如果没有可用任务则返回空列表。

```json
[
  {
    "taskType": "my_task",
    "status": "IN_PROGRESS",
    "taskId": "task-uuid-1",
    "workflowInstanceId": "workflow-uuid-1",
    "inputData": {"key": "value1"}
  },
  {
    "taskType": "my_task",
    "status": "IN_PROGRESS",
    "taskId": "task-uuid-2",
    "workflowInstanceId": "workflow-uuid-2",
    "inputData": {"key": "value2"}
  }
]
```

### 更新任务

```
POST /api/tasks
```

更新任务执行的结果。返回任务 ID。

```shell
curl -X POST 'http://localhost:8080/api/tasks' \
  -H 'Content-Type: application/json' \
  -d '{
    "workflowInstanceId": "3a5b8c2d-1234-5678-9abc-def012345678",
    "taskId": "a1b2c3d4-5678-90ab-cdef-111111111111",
    "status": "COMPLETED",
    "outputData": {
      "result": "processed successfully",
      "recordCount": 42
    }
  }'
```

**请求体字段：**

| 字段 | 说明 | 必填 |
|---|---|---|
| `workflowInstanceId` | 工作流执行 ID | 是 |
| `taskId` | 任务 ID | 是 |
| `status` | `IN_PROGRESS`、`COMPLETED`、`FAILED` 或 `FAILED_WITH_TERMINAL_ERROR` | 是 |
| `outputData` | 输出数据的 JSON 映射 | 否 |
| `reasonForIncompletion` | 当 `status` 为 `FAILED` 或 `FAILED_WITH_TERMINAL_ERROR` 时记录在任务上的自由文本说明。截断为 500 个字符。参见 [理解 reasonForIncompletion](../../devguide/how-tos/Workflows/debugging-workflows.md#understanding-reasonforincompletion)。 | 否 |
| `callbackAfterSeconds` | 回调延迟 — 任务在此时间之后会重新入队 | 否 |
| `logs` | 要追加的日志条目列表 | 否 |

**响应** `200 OK` — 以纯文本返回任务 ID。

### 更新任务 V2

```
POST /api/tasks/update-v2
```

更新任务并返回**下一个可用任务**以便处理 — 将更新和轮询合并在一次调用中。如果没有下一个可用任务，返回 `204 No Content`。

```shell
curl -X POST 'http://localhost:8080/api/tasks/update-v2' \
  -H 'Content-Type: application/json' \
  -d '{
    "workflowInstanceId": "3a5b8c2d-1234-5678-9abc-def012345678",
    "taskId": "a1b2c3d4-5678-90ab-cdef-111111111111",
    "status": "COMPLETED",
    "outputData": {"result": "done"}
  }'
```

**响应** `200 OK` — 返回下一个任务对象，如果没有任务排队则返回 `204 No Content`。

### 按引用名更新任务

```
POST /api/tasks/{workflowId}/{taskRefName}/{status}?workerid=
```

使用工作流 ID 和任务引用名（而不是任务 ID）来更新任务。这对于从外部系统完成 WAIT 或 HUMAN 任务非常有用。

| 参数 | 说明 | 必填 |
|---|---|---|
| `workflowId` | 工作流执行 ID | 是 |
| `taskRefName` | 工作流中的任务引用名 | 是 |
| `status` | `IN_PROGRESS`、`COMPLETED`、`FAILED` 或 `FAILED_WITH_TERMINAL_ERROR` | 是 |
| `workerid` | 工作者标识 | 否 |

请求体：输出数据的 JSON 映射。

```shell
# Complete a WAIT task with output data
curl -X POST 'http://localhost:8080/api/tasks/3a5b8c2d.../wait_for_approval/COMPLETED' \
  -H 'Content-Type: application/json' \
  -d '{"approved": true, "approver": "jane@example.com"}'
```

**响应** `200 OK` — 无响应体。

### 按引用名更新任务（同步）

```
POST /api/tasks/{workflowId}/{taskRefName}/{status}/sync?workerid=
```

与上述相同，但在任务更新被处理后返回**已更新的工作流**。适用于更新任务后需要立即获取工作流状态的同步执行模式。

```shell
curl -X POST 'http://localhost:8080/api/tasks/3a5b8c2d.../wait_for_signal/COMPLETED/sync' \
  -H 'Content-Type: application/json' \
  -d '{"signal": "proceed"}'
```

**响应** `200 OK` — 返回完整的工作流执行对象。

---

## 任务日志

### 添加任务日志

```
POST /api/tasks/{taskId}/log
```

为任务添加一条执行日志。请求体：以纯字符串形式给出的日志消息。

```shell
curl -X POST 'http://localhost:8080/api/tasks/a1b2c3d4.../log' \
  -H 'Content-Type: text/plain' \
  -d 'Processing started for batch #42'
```

**响应** `200 OK` — 无响应体。

### 获取任务日志

```
GET /api/tasks/{taskId}/log
```

返回任务的执行日志。如果不存在日志，返回 `204 No Content`。

```shell
curl 'http://localhost:8080/api/tasks/a1b2c3d4.../log'
```

**响应** `200 OK`

```json
[
  {
    "log": "Processing started for batch #42",
    "taskId": "a1b2c3d4-5678-90ab-cdef-111111111111",
    "createdTime": 1700000001000
  },
  {
    "log": "Batch #42 completed: 100 records processed",
    "taskId": "a1b2c3d4-5678-90ab-cdef-111111111111",
    "createdTime": 1700000003000
  }
]
```

---

## 队列管理

| 端点 | 方法 | 说明 |
|---|---|---|
| `/queue/all` | `GET` | 获取所有队列的待处理任务计数 |
| `/queue/all/verbose` | `GET` | 获取包含每分片计数的详细队列信息 |
| `/queue/size` | `GET` | 获取特定任务类型的队列大小 |
| `/queue/sizes` | `GET` | *（已弃用）* 获取任务类型的队列大小。请改用 `/queue/size`。 |
| `/queue/requeue/{taskType}` | `POST` | 重新入队给定类型的待处理任务 |

### 获取队列大小

```
GET /api/tasks/queue/size?taskType=&domain=&isolationGroupId=&executionNamespace=
```

返回特定任务类型的队列深度，可选地按域和隔离组过滤。

```shell
curl 'http://localhost:8080/api/tasks/queue/size?taskType=my_task'
```

**响应** `200 OK`

```json
5
```

### 获取所有队列大小

```
GET /api/tasks/queue/all
```

返回所有队列的任务类型到待处理计数的映射。

```shell
curl 'http://localhost:8080/api/tasks/queue/all'
```

**响应** `200 OK`

```json
{
  "my_task": 5,
  "http_task": 0,
  "email_task": 12
}
```

### 获取所有队列详情（详细）

```
GET /api/tasks/queue/all/verbose
```

返回包含每分片计数的详细队列信息。

```shell
curl 'http://localhost:8080/api/tasks/queue/all/verbose'
```

**响应** `200 OK`

```json
{
  "my_task": {
    "size": 5,
    "shards": {"0": 3, "1": 2}
  }
}
```

### 重新入队待处理任务

```
POST /api/tasks/queue/requeue/{taskType}
```

重新入队指定类型的所有待处理任务。在工作者出现问题后做恢复时非常有用。

```shell
curl -X POST 'http://localhost:8080/api/tasks/queue/requeue/my_task'
```

**响应** `200 OK` — 返回重新入队的任务数。

---

## 轮询数据

### 获取任务类型的轮询数据

```
GET /api/tasks/queue/polldata?taskType=
```

返回给定任务类型的最后轮询数据 — 便于监控工作者的健康状态与活动。

```shell
curl 'http://localhost:8080/api/tasks/queue/polldata?taskType=my_task'
```

**响应** `200 OK`

```json
[
  {
    "queueName": "my_task",
    "domain": null,
    "workerId": "worker-host-1",
    "lastPollTime": 1700000005000
  }
]
```

### 获取所有任务类型的轮询数据

```
GET /api/tasks/queue/polldata/all
```

返回所有任务类型的最后轮询数据。

```shell
curl 'http://localhost:8080/api/tasks/queue/polldata/all'
```

**响应** `200 OK` — 返回所有任务类型的轮询数据对象列表（与上文相同格式）。

---

## 搜索任务 { #search-tasks }

所有搜索端点支持相同的查询参数：

| 参数 | 说明 | 默认值 |
|---|---|---|
| `start` | 分页偏移 | `0` |
| `size` | 结果数量 | `100` |
| `sort` | 排序顺序：`<field>:ASC` 或 `<field>:DESC` | — |
| `freeText` | 全文搜索查询 | `*` |
| `query` | SQL 风格的 where 子句 | — |

### 搜索（摘要）

```
GET /api/tasks/search?start=0&size=100&sort=&freeText=&query=
```

返回 `SearchResult<TaskSummary>` — 轻量结果。

```shell
# Find failed tasks for a specific workflow type
curl 'http://localhost:8080/api/tasks/search?query=workflowType%3D%27order_processing%27+AND+status%3D%27FAILED%27&size=10'

# Free-text search
curl 'http://localhost:8080/api/tasks/search?freeText=timeout'
```

**响应** `200 OK`

```json
{
  "totalHits": 3,
  "results": [
    {
      "taskId": "task-uuid",
      "taskType": "my_task",
      "referenceTaskName": "my_task_ref",
      "workflowId": "workflow-uuid",
      "workflowType": "order_processing",
      "status": "FAILED",
      "startTime": "2024-01-15T10:30:00Z",
      "updateTime": "2024-01-15T10:30:05Z",
      "executionTime": 5000
    }
  ]
}
```

### 搜索 V2（完整）

```
GET /api/tasks/search-v2?start=0&size=100&sort=&freeText=&query=
```

返回 `SearchResult<Task>` — 包含输入/输出数据的完整任务对象。

---

## 外部存储

```
GET /api/tasks/externalstoragelocation?path=&operation=&payloadType=
```

获取外部任务负载存储的 URI。参见 [外部负载存储](../advanced/externalpayloadstorage.md)。

```shell
curl 'http://localhost:8080/api/tasks/externalstoragelocation?path=task/output&operation=WRITE&payloadType=TASK_OUTPUT'
```

**响应** `200 OK`

```json
{
  "uri": "s3://conductor-payloads/task/output/...",
  "path": "task/output/..."
}
```
