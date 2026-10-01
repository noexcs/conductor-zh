---
description: "Conductor 批量操作 API — 批量暂停、恢复、重启、重试、终止、移除和搜索工作流。"
---

# 批量操作 API

批量操作 API 允许你在单个请求中对多个工作流执行管理工作流的操作。所有端点都使用基础路径 `/api/workflow/bulk`。

每个端点都在请求体中接受工作流 ID 列表，并返回 `BulkResponse`：

```json
{
  "bulkSuccessfulResults": ["workflow-id-1", "workflow-id-2"],
  "bulkErrorResults": {
    "workflow-id-3": "Workflow is not in a running state"
  }
}
```

这些操作是**尽力而为**的 — 每个工作流独立处理。如果其中一个失败，其余的仍会继续执行。

## 端点

| 端点 | 方法 | 说明 |
|---|---|---|
| `/bulk/pause` | `PUT` | 暂停多个工作流 |
| `/bulk/resume` | `PUT` | 恢复多个已暂停的工作流 |
| `/bulk/restart` | `POST` | 重启多个已完成的工作流 |
| `/bulk/retry` | `POST` | 重试多个工作流中最后一个失败的任务 |
| `/bulk/terminate` | `POST` | 终止多个运行中的工作流 |
| `/bulk/remove` | `DELETE` | 从系统中移除多个工作流 |
| `/bulk/terminate-remove` | `DELETE` | 终止并移除多个工作流 |
| `/bulk/search` | `POST` | 按 ID 搜索/获取多个工作流 |

### 批量暂停

```
PUT /api/workflow/bulk/pause
```

```shell
curl -X PUT 'http://localhost:8080/api/workflow/bulk/pause' \
  -H 'Content-Type: application/json' \
  -d '["workflow-id-1", "workflow-id-2", "workflow-id-3"]'
```

**响应** `200 OK`

```json
{
  "bulkSuccessfulResults": ["workflow-id-1", "workflow-id-2"],
  "bulkErrorResults": {
    "workflow-id-3": "Workflow is already paused"
  }
}
```

### 批量恢复

```
PUT /api/workflow/bulk/resume
```

```shell
curl -X PUT 'http://localhost:8080/api/workflow/bulk/resume' \
  -H 'Content-Type: application/json' \
  -d '["workflow-id-1", "workflow-id-2"]'
```

**响应** `200 OK` — 返回 `BulkResponse`。

### 批量重启

```
POST /api/workflow/bulk/restart?useLatestDefinitions=false
```

| 参数 | 说明 | 默认值 |
|---|---|---|
| `useLatestDefinitions` | 使用最新的工作流定义和任务定义 | `false` |

```shell
curl -X POST 'http://localhost:8080/api/workflow/bulk/restart?useLatestDefinitions=true' \
  -H 'Content-Type: application/json' \
  -d '["workflow-id-1", "workflow-id-2"]'
```

**响应** `200 OK` — 返回 `BulkResponse`。

### 批量重试

```
POST /api/workflow/bulk/retry
```

为每个工作流重试最后一个失败的任务。

```shell
curl -X POST 'http://localhost:8080/api/workflow/bulk/retry' \
  -H 'Content-Type: application/json' \
  -d '["workflow-id-1", "workflow-id-2"]'
```

**响应** `200 OK` — 返回 `BulkResponse`。

### 批量终止

```
POST /api/workflow/bulk/terminate?reason=
```

| 参数 | 说明 | 必填 |
|---|---|---|
| `reason` | 终止原因 | 否 |

```shell
curl -X POST 'http://localhost:8080/api/workflow/bulk/terminate?reason=batch+cleanup' \
  -H 'Content-Type: application/json' \
  -d '["workflow-id-1", "workflow-id-2", "workflow-id-3"]'
```

**响应** `200 OK` — 返回 `BulkResponse`。

### 批量移除

```
DELETE /api/workflow/bulk/remove?archiveWorkflow=true
```

| 参数 | 说明 | 默认值 |
|---|---|---|
| `archiveWorkflow` | 移除前归档 | `true` |

```shell
curl -X DELETE 'http://localhost:8080/api/workflow/bulk/remove' \
  -H 'Content-Type: application/json' \
  -d '["workflow-id-1", "workflow-id-2"]'
```

!!! warning
    这会永久删除工作流执行数据。

**响应** `200 OK` — 返回 `BulkResponse`。

### 批量终止并移除

```
DELETE /api/workflow/bulk/terminate-remove?reason=&archiveWorkflow=true
```

一次调用中终止运行中的工作流并将其移除。

| 参数 | 说明 | 默认值 |
|---|---|---|
| `reason` | 终止原因 | — |
| `archiveWorkflow` | 移除前归档 | `true` |

```shell
curl -X DELETE 'http://localhost:8080/api/workflow/bulk/terminate-remove?reason=decommissioned' \
  -H 'Content-Type: application/json' \
  -d '["workflow-id-1", "workflow-id-2"]'
```

**响应** `200 OK` — 返回 `BulkResponse`。

### 批量搜索

```
POST /api/workflow/bulk/search?includeTasks=true
```

在单次调用中按 ID 获取多个工作流。与其他批量端点不同，此端点返回工作流对象，而不是 `BulkResponse`。

| 参数 | 说明 | 默认值 |
|---|---|---|
| `includeTasks` | 包含任务详情 | `true` |

```shell
curl -X POST 'http://localhost:8080/api/workflow/bulk/search?includeTasks=false' \
  -H 'Content-Type: application/json' \
  -d '["workflow-id-1", "workflow-id-2"]'
```

**响应** `200 OK` — 返回 `BulkResponse`，其中 `bulkSuccessfulResults` 包含完整的工作流对象。
