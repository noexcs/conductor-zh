---
description: "Conductor 工作流 API — 通过 REST 管理工作流执行，包括暂停、恢复、重试、重启、重跑、终止、搜索，以及测试工作流。"
---

# 工作流 API

工作流 API 管理工作流执行。所有端点都使用基础路径 `/api/workflow`。

工作流响应是运行时对象；其详细契约为 [Workflow.json](../configuration/schemas.md#runtime-objects)。已注册的蓝图为 [WorkflowDef.json](../configuration/schemas.md#definition-objects)。

启动工作流参见 [启动工作流 API](startworkflow.md)。

## 工作流消息

`POST /api/workflow/{workflowId}/messages` 将任意 JSON 对象推入运行中工作流的消息队列。仅当 `conductor.workflow-message-queue.enabled=true` 时此端点才可用；功能禁用时，控制器不会被注册，端点返回 `404 Not Found`。

```shell
curl -X POST 'http://localhost:8080/api/workflow/3a5b8c2d-1234-5678-9abc-def012345678/messages' \
  -H 'Content-Type: application/json' \
  -d '{"text":"hello"}'
```

**响应** `200 OK` — 纯文本的生成消息 ID。

| 状态 | 条件 |
|---|---|
| `404 Not Found` | WMQ 功能已禁用，或工作流不存在。 |
| `409 Conflict` | 工作流不处于 `RUNNING` 状态，包括与推送相竞态的状态变更。 |
| `429 Too Many Requests` | 工作流队列已达到 `maxQueueSize`。 |

成功推送后，Conductor 会触发一次立即的工作流求值，使等待中的 `PULL_WORKFLOW_MESSAGES` 任务得以恢复。配置与消费参见 [工作流消息队列](../../wmq/workflow-message-queue.md) 和 [拉取工作流消息任务](../configuration/workflowdef/systemtasks/pull-workflow-messages-task.md)。

## 获取工作流

| 端点 | 方法 | 说明 |
|---|---|---|
| `/{workflowId}` | `GET` | 按 ID 获取工作流执行 |
| `/{workflowId}/status` | `GET` | 获取轻量工作流状态摘要 |
| `/{workflowId}/tasks` | `GET` | 获取工作流执行的任务（分页） |
| `/running/{name}` | `GET` | 按类型获取运行中的工作流 ID |
| `/{name}/correlated/{correlationId}` | `GET` | 按关联 ID 获取工作流 |
| `/{name}/correlated` | `POST` | 按多个关联 ID 获取工作流 |

### 按 ID 获取工作流

```
GET /api/workflow/{workflowId}?includeTasks=true
```

| 参数 | 说明 | 默认值 |
|---|---|---|
| `workflowId` | 工作流执行 ID | — |
| `includeTasks` | 在响应中包含任务详情 | `true` |

```shell
curl 'http://localhost:8080/api/workflow/3a5b8c2d-1234-5678-9abc-def012345678'
```

**响应** `200 OK`

```json
{
  "workflowId": "3a5b8c2d-1234-5678-9abc-def012345678",
  "workflowName": "order_processing",
  "workflowVersion": 1,
  "status": "COMPLETED",
  "startTime": 1700000000000,
  "endTime": 1700000005000,
  "input": {"orderId": "ORD-123"},
  "output": {"paymentId": "PAY-456"},
  "tasks": [
    {
      "taskId": "task-uuid",
      "taskType": "HTTP",
      "referenceTaskName": "validate",
      "status": "COMPLETED",
      "outputData": {"response": {"statusCode": 200}}
    }
  ],
  "correlationId": "order-123"
}
```

**状态与失败字段**

| 字段 | 说明 |
|---|---|
| `status` | `RUNNING`、`PAUSED`、`COMPLETED`、`FAILED`、`TIMED_OUT` 或 `TERMINATED`。 |
| `reasonForIncompletion` | 工作流未完成即停止的原因：失败任务的原因、超时消息，或传给 terminate 的原因。运行期间为空；由重试、重启和重跑清除。参见 [理解 reasonForIncompletion](../../devguide/how-tos/Workflows/debugging-workflows.md#understanding-reasonforincompletion)。 |
| `failedReferenceTaskNames` | 失败任务的引用名。 |
| `failedTaskNames` | 失败任务的定义名。 |
| `lastRetriedTime` | 最近一次重试的纪元毫秒数；从未重试时为 `0`。 |
| `reRunFromWorkflowId` | 当此执行是从另一个执行重跑时设置。 |
| `parentWorkflowId`、`parentWorkflowTaskId` | 子工作流上存在：父执行及其 `SUB_WORKFLOW` 任务。 |
| `event` | 当事件处理器启动工作流时，启动该工作流的事件名称。 |

### 获取工作流状态摘要

```http
GET /api/workflow/{workflowId}/status?includeOutput=false&includeVariables=false
```

此端点返回 `WorkflowStatus`，即轻量摘要。`includeOutput` 和 `includeVariables` 默认均为 `false`；仅在需要相应数据时才将其设为 `true`。

### 获取工作流的任务

```
GET /api/workflow/{workflowId}/tasks?start=0&count=15&status=
```

返回工作流执行的分页任务列表。

| 参数 | 说明 | 默认值 |
|---|---|---|
| `start` | 分页偏移 | `0` |
| `count` | 结果数量 | `15` |
| `status` | 按任务状态过滤（可指定多个） | 所有状态 |

```shell
# Get first 10 tasks
curl 'http://localhost:8080/api/workflow/3a5b8c2d.../tasks?count=10'

# Get only failed tasks
curl 'http://localhost:8080/api/workflow/3a5b8c2d.../tasks?status=FAILED'
```

**响应** `200 OK`

```json
{
  "totalHits": 5,
  "results": [
    {
      "taskId": "task-uuid",
      "taskType": "HTTP",
      "referenceTaskName": "validate",
      "status": "COMPLETED"
    }
  ]
}
```

### 获取运行中的工作流

```
GET /api/workflow/running/{name}?version=1&startTime=&endTime=
```

返回给定类型的运行中工作流的 ID 列表。

| 参数 | 说明 | 默认值 |
|---|---|---|
| `name` | 工作流名称 | — |
| `version` | 工作流版本 | `1` |
| `startTime` | 按开始时间过滤（纪元毫秒） | — |
| `endTime` | 按结束时间过滤（纪元毫秒） | — |

```shell
curl 'http://localhost:8080/api/workflow/running/order_processing?version=1'
```

**响应** `200 OK`

```json
["3a5b8c2d-1234-...", "7f8e9d0c-5678-..."]
```

### 按关联 ID 获取工作流

```
GET /api/workflow/{name}/correlated/{correlationId}?includeClosed=false&includeTasks=false
```

| 参数 | 说明 | 默认值 |
|---|---|---|
| `includeClosed` | 包含已完成/已终止的工作流 | `false` |
| `includeTasks` | 包含任务详情 | `false` |

```shell
curl 'http://localhost:8080/api/workflow/order_processing/correlated/order-123?includeClosed=true'
```

### 按多个关联 ID 获取工作流

```
POST /api/workflow/{name}/correlated?includeClosed=false&includeTasks=false
```

```shell
curl -X POST 'http://localhost:8080/api/workflow/order_processing/correlated?includeClosed=true' \
  -H 'Content-Type: application/json' \
  -d '["order-123", "order-456", "order-789"]'
```

**响应** `200 OK` — 关联 ID 到工作流列表的映射。

---

## 管理工作流

| 端点 | 方法 | 说明 |
|---|---|---|
| `/{workflowId}/pause` | `PUT` | 暂停工作流 |
| `/{workflowId}/resume` | `PUT` | 恢复已暂停的工作流 |
| `/{workflowId}/restart` | `POST` | 从头重启已完成的工作流 |
| `/{workflowId}/retry` | `POST` | 重试最后一个失败的任务 |
| `/{workflowId}/rerun` | `POST` | 从指定任务重跑 |
| `/{workflowId}/skiptask/{taskReferenceName}` | `PUT` | 跳过运行中工作流中的任务 |
| `/{workflowId}/resetcallbacks` | `POST` | 重置 SIMPLE 任务的回调时间 |
| `/decide/{workflowId}` | `PUT` | 触发工作流的决策器 |
| `/{workflowId}` | `DELETE` | 终止运行中的工作流 |
| `/{workflowId}/remove` | `DELETE` | 从系统中移除工作流 |
| `/{workflowId}/terminate-remove` | `DELETE` | 一次调用中终止并移除 |

### 暂停

```
PUT /api/workflow/{workflowId}/pause
```

暂停工作流。在恢复之前，不会调度更多任务。当前正在运行的任务**不受**影响。

```shell
curl -X PUT 'http://localhost:8080/api/workflow/3a5b8c2d.../pause'
```

### 恢复

```
PUT /api/workflow/{workflowId}/resume
```

```shell
curl -X PUT 'http://localhost:8080/api/workflow/3a5b8c2d.../resume'
```

### 重启

```
POST /api/workflow/{workflowId}/restart?useLatestDefinitions=false
```

从头重启已完成的工作流。当前的执行历史会被清除。

| 参数 | 说明 | 默认值 |
|---|---|---|
| `useLatestDefinitions` | 使用最新的工作流定义和任务定义 | `false` |

```shell
curl -X POST 'http://localhost:8080/api/workflow/3a5b8c2d.../restart'
```

### 重试

```
POST /api/workflow/{workflowId}/retry?resumeSubworkflowTasks=false
```

重试工作流中最后一个失败的任务。

| 参数 | 说明 | 默认值 |
|---|---|---|
| `resumeSubworkflowTasks` | 同时恢复失败的子工作流任务 | `false` |

```shell
curl -X POST 'http://localhost:8080/api/workflow/3a5b8c2d.../retry'
```

### 重跑

```
POST /api/workflow/{workflowId}/rerun
```

从指定任务重跑已完成的工作流。

```shell
curl -X POST 'http://localhost:8080/api/workflow/3a5b8c2d.../rerun' \
  -H 'Content-Type: application/json' \
  -d '{
    "reRunFromWorkflowId": "3a5b8c2d...",
    "workflowInput": {"orderId": "ORD-999"},
    "reRunFromTaskId": "task-uuid",
    "taskInput": {"override": true}
  }'
```

### 跳过任务

```
PUT /api/workflow/{workflowId}/skiptask/{taskReferenceName}
```

跳过运行中工作流里的任务并继续向前执行。可选地提供更新后的输入/输出：

```shell
curl -X PUT 'http://localhost:8080/api/workflow/3a5b8c2d.../skiptask/validate_ref' \
  -H 'Content-Type: application/json' \
  -d '{
    "taskInput": {},
    "taskOutput": {"skipped": true, "reason": "manual override"}
  }'
```

### 重置回调

```
POST /api/workflow/{workflowId}/resetcallbacks
```

将所有非终态 SIMPLE 任务的回调时间重置为 0，使它们立即被重新求值。

### 决策

```
PUT /api/workflow/decide/{workflowId}
```

手动触发工作流的决策器。决策器对工作流状态进行求值并调度下一个任务。这通常是自动执行的 — 此端点用于调试。

### 终止

```
DELETE /api/workflow/{workflowId}?reason=
```

| 参数 | 说明 | 必填 |
|---|---|---|
| `reason` | 终止原因 | 否 |

```shell
curl -X DELETE 'http://localhost:8080/api/workflow/3a5b8c2d...?reason=cancelled+by+user'
```

### 移除

```
DELETE /api/workflow/{workflowId}/remove?archiveWorkflow=true
```

| 参数 | 说明 | 默认值 |
|---|---|---|
| `archiveWorkflow` | 移除前归档 | `true` |

!!! warning
    这会永久删除工作流执行数据。请谨慎使用。

### 终止并移除

```
DELETE /api/workflow/{workflowId}/terminate-remove?reason=&archiveWorkflow=true
```

一次调用中终止运行中的工作流，并将其从系统中移除。

---

## 搜索工作流 { #search-workflows }

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
GET /api/workflow/search?start=0&size=100&sort=&freeText=&query=
```

返回 `SearchResult<WorkflowSummary>` — 不含完整工作流详情的轻量结果。

```shell
# Find completed workflows of a specific type
curl 'http://localhost:8080/api/workflow/search?query=workflowType%3D%27order_processing%27+AND+status%3D%27COMPLETED%27&size=10'

# Free-text search
curl 'http://localhost:8080/api/workflow/search?freeText=order-123'
```

**响应** `200 OK`

```json
{
  "totalHits": 42,
  "results": [
    {
      "workflowType": "order_processing",
      "version": 1,
      "workflowId": "3a5b8c2d...",
      "correlationId": "order-123",
      "startTime": "2024-01-15T10:30:00Z",
      "updateTime": "2024-01-15T10:30:05Z",
      "endTime": "2024-01-15T10:30:05Z",
      "status": "COMPLETED",
      "executionTime": 5000
    }
  ]
}
```

### 搜索 V2（完整）

```
GET /api/workflow/search-v2
```

参数与 search 相同，但返回 `SearchResult<Workflow>` — 包含任务详情的完整工作流对象。

### 按任务搜索 { #search-by-tasks }

```
GET /api/workflow/search-by-tasks
```

基于任务级参数搜索工作流。返回 `SearchResult<WorkflowSummary>`。

### 按任务搜索 V2

```
GET /api/workflow/search-by-tasks-v2
```

返回包含完整工作流对象的 `SearchResult<Workflow>`。

### 查询语法 { #query-syntax }

`query` 参数支持 SQL 风格的表达式：

| 示例 | 说明 |
|---|---|
| `workflowType = 'order_processing'` | 按工作流类型过滤 |
| `status = 'FAILED'` | 按状态过滤 |
| `startTime > 1700000000000` | 按开始时间过滤（纪元毫秒） |
| `workflowType = 'order_processing' AND status = 'COMPLETED'` | 组合条件 |

`freeText` 参数支持 Elasticsearch 查询语法：

| 示例 | 说明 |
|---|---|
| `workflowType:"order_processing"` | 匹配工作流类型 |
| `order-123` | 匹配任意字段 |

---

## 测试工作流

```
POST /api/workflow/test
```

使用模拟数据测试工作流执行，而不实际运行它。适用于在部署前验证工作流定义和任务接线。

```shell
curl -X POST 'http://localhost:8080/api/workflow/test' \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "my_workflow",
    "version": 1,
    "workflowDef": {...},
    "taskRefToMockOutput": {
      "my_task_ref": [{
        "status": "COMPLETED",
        "output": {"key": "mocked_value"}
      }]
    }
  }'
```

**响应** `200 OK` — 返回带模拟任务输出的仿真工作流执行。

---

## 外部存储

```
GET /api/workflow/externalstoragelocation?path=&operation=&payloadType=
```

获取外部负载存储的 URI。参见 [外部负载存储](../advanced/externalpayloadstorage.md)。
