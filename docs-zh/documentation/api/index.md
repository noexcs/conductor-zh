---
description: "Conductor REST API 参考 — 用于工作流编排和 Conductor Agents 的端点，涵盖定义、执行管理、任务、事件和持久化 Agent 控制。"
---

# API 参考

Conductor 为工作流定义与执行、工作者任务、调度、事件、文件、批量操作、任务域（Task Domains）以及 Conductor Agents 提供公共 REST API。完整的部署特定接口（包括管理和 UI 支持端点）可在 Swagger 中查看。

## 参考约定

- **可用性门控：** 带有某属性标注的路由或任务，只有在该服务器属性启用时才会被注册。调度器端点需要调度器条件；AI 任务与 Agent 能力需要各自的 AI 运行时配置。
- **弃用：** 已弃用的路由和任务类型仅保留用于迁移指引。新集成请使用文档中给出的替代方案。
- **定义与运行时：** 工作流/任务定义是可复用的蓝图；工作流/任务对象是单独的执行记录。字段级契约参见 [Schemas](../configuration/schemas.md) 及其直接的 source-schema 链接。

## 基础 URL

所有 API 端点都相对于你的 Conductor 服务器的基础 URL：

```
http://localhost:8080/api/
```

例如，要列出所有工作流定义：

```shell
curl http://localhost:8080/api/metadata/workflow
```

如果你的 Conductor 服务器运行在其他主机或端口上，请相应替换 `localhost:8080`。

## 认证

Conductor OSS 默认不需要认证。所有 API 端点都是开放的。如果你需要保护 Conductor 实例，可以通过反向代理（如 Nginx、Envoy）添加认证，或在 Spring Boot 中实现自定义安全过滤器。

## 内容类型

所有请求体和响应体都使用 JSON。对于带请求体的请求，请设置以下请求头：

```
Content-Type: application/json
```

少数端点返回纯文本（例如启动时返回工作流 ID）。这些端点在各自文档中都有说明。

## 常见响应码

| 状态码 | 说明 |
|---|---|
| `200 OK` | 请求成功。响应体包含结果。 |
| `204 No Content` | 请求成功但没有响应体（例如轮询时没有可用任务）。 |
| `400 Bad Request` | 无效请求 — 请检查你的请求体或参数。 |
| `404 Not Found` | 请求的资源（工作流、任务、定义）不存在。 |
| `409 Conflict` | 与当前状态冲突（例如尝试恢复一个未暂停的工作流）。 |
| `500 Internal Server Error` | 服务器端错误。请检查 Conductor 服务器日志。 |

### 错误响应格式

发生错误时，响应体包含：

```json
{
  "status": 400,
  "message": "Workflow definition is not valid",
  "instance": "conductor-server",
  "retryable": false
}
```

## 快速开始

注册一个工作流定义、启动它并检查其状态 — 只需三条命令：

```shell
# 1. Register a workflow definition
curl -X POST 'http://localhost:8080/api/metadata/workflow' \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "hello_workflow",
    "version": 1,
    "tasks": [
      {
        "name": "hello_task",
        "taskReferenceName": "hello_ref",
        "type": "HTTP",
        "inputParameters": {
          "http_request": {
            "uri": "https://jsonplaceholder.typicode.com/posts/1",
            "method": "GET"
          }
        }
      }
    ],
    "schemaVersion": 2
  }'

# 2. Start a workflow execution
WORKFLOW_ID=$(curl -s -X POST 'http://localhost:8080/api/workflow/hello_workflow' \
  -H 'Content-Type: application/json' \
  -d '{}')
echo "Started workflow: $WORKFLOW_ID"

# 3. Check workflow status
curl "http://localhost:8080/api/workflow/$WORKFLOW_ID"
```

## API 章节

| 章节 | 基础路径 | 说明 |
|---|---|---|
| **[元数据](metadata.md)** | `/api/metadata` | 注册、更新、校验和删除工作流定义与任务定义 |
| **[启动工作流](startworkflow.md)** | `/api/workflow` | 异步、同步或使用动态定义启动工作流 |
| **[工作流](workflow.md)** | `/api/workflow` | 管理执行：获取状态、暂停、恢复、重试、重启、终止、搜索 |
| **[任务](task.md)** | `/api/tasks` | 轮询任务、更新结果、管理队列、查看日志、搜索 |
| **[批量操作](bulk.md)** | `/api/workflow/bulk` | 批量暂停、恢复、重启、重试、终止或删除工作流 |
| **[事件处理器](eventhandlers.md)** | `/api/event` | 创建和管理基于事件的工作流触发器 |
| **[文件](files.md)** | `/api/files` | 创建工作流作用域的文件句柄并交换签名上传/下载 URL；需要文件存储 |
| **[任务域](taskdomains.md)** | — | 在运行时将任务路由到特定的工作者池 |
| **[调度器](scheduler.md)** | `/api/scheduler` | 创建、搜索、暂停、恢复并批量管理调度；需要 `conductor.scheduler.enabled=true` |
| **[Conductor Agents](agents.md)** | `/api/agent` | 编译、部署、启动、观察并控制由 SDK 编写的持久化 Agent；需要 `conductor.integrations.ai.enabled=true` |

## Swagger UI

`http://localhost:8080/swagger-ui/index.html` 处的 Swagger UI 提供了一个交互式 API 浏览器，你可以直接在浏览器中试用端点。

## SDK

要进行编程访问，请使用官方的 [Conductor SDK](../clientsdks/index.md) 之一，它们以 Java、Python、Go、JavaScript、C#、Ruby 和 Rust 的原生接口封装了这些 REST API。
