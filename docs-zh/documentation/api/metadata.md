---
description: "Conductor 元数据 API — 注册、更新、校验和删除工作流定义与任务定义。通过 REST 管理你的编排蓝图。"
---

# 元数据 API

元数据端点管理定义对象。标准的 `WorkflowDef.json` 和 `TaskDef.json` 契约参见 [Schemas](../configuration/schemas.md)。

元数据 API 管理工作流定义和任务定义 — 即 Conductor 用来编排执行的蓝图。所有端点都使用基础路径 `/api/metadata`。

## 工作流定义

| 端点 | 方法 | 说明 |
|---|---|---|
| `/metadata/workflow` | `GET` | 获取所有工作流定义 |
| `/metadata/workflow` | `POST` | 创建新的工作流定义 |
| `/metadata/workflow` | `PUT` | 创建或更新工作流定义（批量） |
| `/metadata/workflow/{name}` | `GET` | 按名称获取工作流定义 |
| `/metadata/workflow/{name}/{version}` | `DELETE` | 按名称和版本删除工作流定义 |
| `/metadata/workflow/validate` | `POST` | 校验工作流定义而不保存 |
| `/metadata/workflow/names-and-versions` | `GET` | 获取所有工作流的名称和版本（不含定义体） |
| `/metadata/workflow/names` | `GET` | 仅获取去重后的工作流名称 |
| `/metadata/workflow/{name}/versions` | `GET` | 获取单个工作流的轻量版本摘要 |
| `/metadata/workflow/latest-versions` | `GET` | 仅获取每个工作流定义的最新版本 |

### 获取所有工作流定义

```
GET /api/metadata/workflow
```

返回所有已注册的工作流定义列表。

```shell
curl http://localhost:8080/api/metadata/workflow
```

**响应** `200 OK`

```json
[
  {
    "name": "order_processing",
    "version": 1,
    "tasks": [...],
    "inputParameters": [],
    "outputParameters": {},
    "schemaVersion": 2
  }
]
```

### 创建工作流定义

```
POST /api/metadata/workflow
```

注册新的工作流定义。请求体是一个[工作流定义](../configuration/workflowdef/index.md)。

```shell
curl -X POST 'http://localhost:8080/api/metadata/workflow' \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "my_workflow",
    "version": 1,
    "tasks": [
      {
        "name": "my_task",
        "taskReferenceName": "my_task_ref",
        "type": "SIMPLE"
      }
    ],
    "schemaVersion": 2,
    "ownerEmail": "dev@example.com"
  }'
```

**响应** `200 OK` — 无响应体。

### 创建或更新工作流定义

```
PUT /api/metadata/workflow
```

批量创建或更新工作流定义。请求体是[工作流定义](../configuration/workflowdef/index.md)列表。返回一个 `BulkResponse`，指示每个定义的成功与失败。

```shell
curl -X PUT 'http://localhost:8080/api/metadata/workflow' \
  -H 'Content-Type: application/json' \
  -d '[
    {"name": "workflow_a", "version": 1, "tasks": [...], "schemaVersion": 2},
    {"name": "workflow_b", "version": 1, "tasks": [...], "schemaVersion": 2}
  ]'
```

**响应** `200 OK`

```json
{
  "bulkSuccessfulResults": ["workflow_a", "workflow_b"],
  "bulkErrorResults": {}
}
```

### 按名称获取工作流定义

```
GET /api/metadata/workflow/{name}?version={version}
```

| 参数 | 说明 | 必填 |
|---|---|---|
| `name` | 工作流名称 | 是（路径） |
| `version` | 工作流版本 | 否（默认为最新版本） |

```shell
curl 'http://localhost:8080/api/metadata/workflow/my_workflow?version=1'
```

**响应** `200 OK` — 返回完整的工作流定义 JSON。

### 删除工作流定义

```
DELETE /api/metadata/workflow/{name}/{version}
```

按名称和版本删除工作流定义。**不会**删除与该定义关联的工作流执行。

| 参数 | 说明 | 必填 |
|---|---|---|
| `name` | 工作流名称 | 是（路径） |
| `version` | 工作流版本 | 是（路径） |

```shell
curl -X DELETE 'http://localhost:8080/api/metadata/workflow/my_workflow/1'
```

**响应** `200 OK` — 无响应体。

### 校验工作流定义

```
POST /api/metadata/workflow/validate
```

校验工作流定义而不注册它。适用于 CI/CD 流水线或部署前检查。

```shell
curl -X POST 'http://localhost:8080/api/metadata/workflow/validate' \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "my_workflow",
    "version": 1,
    "tasks": [
      {
        "name": "my_task",
        "taskReferenceName": "my_task_ref",
        "type": "SIMPLE"
      }
    ],
    "schemaVersion": 2
  }'
```

**响应** 有效时为 `200 OK`。无效时为 `400 Bad Request` 并附带错误详情。

### 获取工作流名称和版本

```
GET /api/metadata/workflow/names-and-versions
```

返回工作流名称到其可用版本的轻量映射（不含定义体）。适用于构建 UI 或列出可用工作流。

```shell
curl http://localhost:8080/api/metadata/workflow/names-and-versions
```

**响应** `200 OK`

```json
{
  "order_processing": [
    {"name": "order_processing", "version": 1},
    {"name": "order_processing", "version": 2}
  ],
  "user_onboarding": [
    {"name": "user_onboarding", "version": 1}
  ]
}
```

### 仅获取最新版本

```
GET /api/metadata/workflow/latest-versions
```

仅返回每个工作流定义的最新版本。

```shell
curl http://localhost:8080/api/metadata/workflow/latest-versions
```

**响应** `200 OK` — 返回工作流定义列表（每个工作流名称一个，仅最新版本）。

### 获取不含定义体的名称或版本

```http
GET /api/metadata/workflow/names
GET /api/metadata/workflow/{name}/versions
```

第一个路由返回去重后的工作流名称 JSON 数组。第二个返回指定工作流的轻量 `WorkflowDefSummary` 值。当调用方需要发现数据而无需下载完整定义时，请使用这些路由。

---

## 任务定义

| 端点 | 方法 | 说明 |
|---|---|---|
| `/metadata/taskdefs` | `GET` | 获取所有任务定义 |
| `/metadata/taskdefs` | `POST` | 创建新的任务定义 |
| `/metadata/taskdefs` | `PUT` | 更新任务定义 |
| `/metadata/taskdefs/{taskType}` | `GET` | 按名称获取任务定义 |
| `/metadata/taskdefs/{taskType}` | `DELETE` | 删除任务定义 |

### 获取所有任务定义

```
GET /api/metadata/taskdefs
```

```shell
curl http://localhost:8080/api/metadata/taskdefs
```

**响应** `200 OK`

```json
[
  {
    "name": "my_task",
    "retryCount": 3,
    "retryLogic": "FIXED",
    "retryDelaySeconds": 10,
    "timeoutSeconds": 300,
    "timeoutPolicy": "TIME_OUT_WF",
    "responseTimeoutSeconds": 180
  }
]
```

### 创建任务定义

```
POST /api/metadata/taskdefs
```

注册新的任务定义。请求体是[任务定义](../configuration/taskdef.md)列表。

```shell
curl -X POST 'http://localhost:8080/api/metadata/taskdefs' \
  -H 'Content-Type: application/json' \
  -d '[
    {
      "name": "my_task",
      "retryCount": 3,
      "retryLogic": "FIXED",
      "retryDelaySeconds": 10,
      "timeoutSeconds": 300,
      "timeoutPolicy": "TIME_OUT_WF",
      "responseTimeoutSeconds": 180,
      "ownerEmail": "dev@example.com"
    }
  ]'
```

**响应** `200 OK` — 无响应体。

### 更新任务定义

```
PUT /api/metadata/taskdefs
```

更新已有的任务定义。请求体是单个[任务定义](../configuration/taskdef.md)。

```shell
curl -X PUT 'http://localhost:8080/api/metadata/taskdefs' \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "my_task",
    "retryCount": 5,
    "retryLogic": "EXPONENTIAL_BACKOFF",
    "retryDelaySeconds": 5,
    "timeoutSeconds": 600,
    "timeoutPolicy": "TIME_OUT_WF",
    "responseTimeoutSeconds": 300
  }'
```

**响应** `200 OK` — 无响应体。

### 按名称获取任务定义

```
GET /api/metadata/taskdefs/{taskType}
```

```shell
curl http://localhost:8080/api/metadata/taskdefs/my_task
```

**响应** `200 OK` — 返回任务定义 JSON。

### 删除任务定义

```
DELETE /api/metadata/taskdefs/{taskType}
```

```shell
curl -X DELETE http://localhost:8080/api/metadata/taskdefs/my_task
```

**响应** `200 OK` — 无响应体。
