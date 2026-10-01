---
description: "启动 Conductor 工作流执行 — 通过 REST API 以异步、同步和动态工作流方式执行，附 curl 示例。"
---

# 启动工作流 API

## 启动工作流（异步） { #start-a-workflow-asynchronous }

```
POST /api/workflow
```

异步启动一个新的工作流执行。立即返回工作流 ID。

### 请求体 { #request-body }

| 字段 | 说明 | 必填 |
|---|---|---|
| `name` | 工作流名称（必须已注册） | 是 |
| `version` | 工作流版本 | 否（默认为最新版本） |
| `input` | 工作流输入参数的 JSON 对象 | 否 |
| `correlationId` | 用于关联多次工作流执行的唯一 ID | 否 |
| `taskToDomain` | 任务到域的映射。参见 [任务域](taskdomains.md)。 | 否 |
| `workflowDef` | 用于动态工作流的内联[工作流定义](../configuration/workflowdef/index.md)。参见 [动态工作流](#dynamic-workflows)。 | 否 |
| `externalInputPayloadStoragePath` | 外部负载存储路径。参见 [外部负载存储](../advanced/externalpayloadstorage.md)。 | 否 |
| `priority` | 此工作流内任务的优先级（0–99） | 否 |

### 示例

```shell
curl -X POST 'http://localhost:8080/api/workflow' \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "myWorkflow",
    "version": 1,
    "correlationId": "order-123",
    "priority": 1,
    "input": {
      "customerId": "CUST-456",
      "amount": 99.99
    },
    "taskToDomain": {
      "*": "mydomain"
    }
  }'
```

**响应** `200 OK` — 以纯文本返回工作流 ID：

```
3a5b8c2d-1234-5678-9abc-def012345678
```

### 通过路径参数启动

```
POST /api/workflow/{name}
```

启动工作流的另一种方式 — 在路径中指定名称，并将输入作为请求体传递。

| 参数 | 类型 | 说明 | 必填 |
|---|---|---|---|
| `name` | Path | 工作流名称 | 是 |
| `version` | Query | 工作流版本 | 否 |
| `correlationId` | Query | 关联 ID | 否 |
| `priority` | Query | 优先级 0–99（默认：0） | 否 |

```shell
curl -X POST 'http://localhost:8080/api/workflow/myWorkflow?version=1&correlationId=order-123' \
  -H 'Content-Type: application/json' \
  -d '{"customerId": "CUST-456", "amount": 99.99}'
```

**响应** `200 OK` — 以纯文本返回工作流 ID。

---

## 执行工作流（同步）

```
POST /api/workflow/execute/{name}/{version}
```

启动工作流并**等待完成**（或指定条件）后才返回结果。这样就无需轮询工作流状态。

| 参数 | 类型 | 说明 | 必填 |
|---|---|---|---|
| `name` | Path | 工作流名称 | 是 |
| `version` | Path | 工作流版本（使用 `0` 表示最新版本） | 是 |
| `requestId` | Query | 幂等键 | 否（自动生成） |
| `waitUntilTaskRef` | Query | 要等待的逗号分隔任务引用名 | 否 |
| `waitForSeconds` | Query | 最长等待秒数 | 否（默认：10） |
| `consistency` | Query | 仅为兼容性而接受；在开源 Conductor 中无效果 — 执行始终是持久化的 | 否（默认：`DURABLE`） |
| `returnStrategy` | Query | 当执行阻塞在 Yield 任务上时返回哪个状态：`TARGET_WORKFLOW`（最初启动的工作流）、`BLOCKING_WORKFLOW`（当前阻塞的工作流，可能是子工作流）、`BLOCKING_TASK`（阻塞任务的状态）或 `BLOCKING_TASK_INPUT`（阻塞任务的输入） | 否（默认：`TARGET_WORKFLOW`） |

请求体：StartWorkflowRequest 对象（与[异步启动](#start-a-workflow-asynchronous)相同格式）。

### 示例

```shell
curl -X POST 'http://localhost:8080/api/workflow/execute/my_workflow/1?waitForSeconds=30' \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "my_workflow",
    "version": 1,
    "input": {
      "url": "https://api.example.com/data"
    }
  }'
```

**响应** `200 OK` — 返回工作流执行结果：

```json
{
  "workflowId": "3a5b8c2d-1234-5678-9abc-def012345678",
  "requestId": "req-uuid",
  "status": "COMPLETED",
  "output": {
    "response": {...}
  },
  "tasks": [...]
}
```

### 等待行为

- 如果指定了 `waitUntilTaskRef`，当任一列出的任务到达终态（或遇到 WAIT 任务）时，API 即返回
- 如果工作流在超时前完成，立即返回结果
- 如果达到超时，返回当前工作流状态 — 工作流继续在后台运行
- 子工作流中的 WAIT 任务会被递归检测到

---

## 动态工作流 { #dynamic-workflows }

无需预注册定义即可启动一次性工作流。通过 `workflowDef` 字段内联提供完整的工作流定义。

```shell
curl -X POST 'http://localhost:8080/api/workflow' \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "my_adhoc_workflow",
    "workflowDef": {
      "ownerApp": "my_app",
      "ownerEmail": "owner@example.com",
      "name": "my_adhoc_workflow",
      "version": 1,
      "tasks": [
        {
          "name": "fetch_data",
          "type": "HTTP",
          "taskReferenceName": "fetch_data",
          "inputParameters": {
            "uri": "${workflow.input.uri}",
            "method": "GET"
          },
          "taskDefinition": {
            "name": "fetch_data",
            "retryCount": 0,
            "timeoutSeconds": 3600,
            "timeoutPolicy": "TIME_OUT_WF",
            "responseTimeoutSeconds": 3000
          }
        }
      ]
    },
    "input": {
      "uri": "https://api.example.com/data"
    }
  }'
```

**响应** `200 OK` — 以纯文本返回工作流 ID。

!!! note
    如果某个 `taskDefinition` 已通过元数据 API 注册，则无需将其内联包含在动态工作流定义中。
