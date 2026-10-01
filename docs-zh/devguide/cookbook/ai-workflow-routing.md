---
description: 使用 LLM 和动态 SUB_WORKFLOW 将客户请求路由到多个已批准工作流之一。
---

# 使用 AI 的动态工作流

使用 LLM 为用户的请求选择最合适的工作流，同时保持执行的持久化。LLM 会看到由工作流名称和描述组成的目录，以 JSON 返回其中一个选择，然后动态 `SUB_WORKFLOW` 运行所选的已注册工作流。

目录是刻意设计的：动态 `SUB_WORKFLOW` 只能启动以所选名称注册的工作流定义。保持提示词中的工作流名称与在 Conductor 中注册的子工作流一致；杜撰的名称会在任何子工作流启动之前就失败。

## 示例：路由客户请求

该路由器可以从三个已注册工作流中选择一个。完整可运行的示例见 [`ai/examples/36-ai-workflow-routing.json`](https://github.com/conductor-oss/conductor/blob/main/ai/examples/36-ai-workflow-routing.json) 及其配套的 `36a`–`36c` 子工作流。

| 工作流 | 说明 |
|---|---|
| `ai_route_support_ticket` | 用于产品缺陷、访问问题和故障排查请求。 |
| `ai_route_refund_request` | 用于退货、退款和重复扣款请求。 |
| `ai_route_sales_lead` | 用于定价、采购和企业销售请求。 |

```json
{
  "name": "ai_workflow_router",
  "description": "Select an approved workflow for a customer request",
  "version": 1,
  "schemaVersion": 2,
  "inputParameters": ["request"],
  "tasks": [
    {
      "name": "select_workflow",
      "taskReferenceName": "select_workflow",
      "type": "LLM_CHAT_COMPLETE",
      "inputParameters": {
        "llmProvider": "openai",
        "model": "gpt-4o-mini",
        "messages": [
          {
            "role": "system",
            "message": "You route customer requests to approved workflows. Choose exactly one workflow from this json catalog and return valid json only. Catalog: [{\"workflow\":\"ai_route_support_ticket\",\"description\":\"Product defects, access problems, and troubleshooting.\"},{\"workflow\":\"ai_route_refund_request\",\"description\":\"Returns, refunds, and duplicate charges.\"},{\"workflow\":\"ai_route_sales_lead\",\"description\":\"Pricing, procurement, and enterprise sales.\"}]"
          },
          {
            "role": "user",
            "message": "Customer request: ${workflow.input.request}. Return valid json with workflow and reason."
          }
        ],
        "temperature": 0,
        "maxTokens": 120,
        "jsonOutput": true
      }
    },
    {
      "name": "run_selected_workflow",
      "taskReferenceName": "run_selected_workflow",
      "type": "SUB_WORKFLOW",
      "inputParameters": {
        "request": "${workflow.input.request}",
        "routingReason": "${select_workflow.output.result.reason}"
      },
      "subWorkflowParam": {
        "name": "${select_workflow.output.result.workflow}",
        "version": 1
      }
    }
  ],
  "outputParameters": {
    "selectedWorkflow": "${select_workflow.output.result.workflow}",
    "routingReason": "${select_workflow.output.result.reason}",
    "subWorkflowId": "${run_selected_workflow.output.subWorkflowId}",
    "subWorkflowOutput": "${run_selected_workflow.output}"
  }
}
```

## 注册路由器及其已批准的目的地

在注册或启动路由器之前，先注册每个目的地工作流。对于本地端到端试用，这些最小化的目的地可以让每个分支可见，而无需调用外部系统：

```json
{
  "name": "ai_route_support_ticket",
  "version": 1,
  "schemaVersion": 2,
  "inputParameters": ["request", "routingReason"],
  "tasks": [{"name": "record_ticket", "taskReferenceName": "record_ticket", "type": "NOOP"}]
}
```

创建名为 `ai_route_refund_request` 和 `ai_route_sales_lead` 的等效占位定义，然后注册全部四个定义：

```shell
curl -X POST 'http://localhost:8080/api/metadata/workflow' -H 'Content-Type: application/json' -d @ai_route_support_ticket.json
curl -X POST 'http://localhost:8080/api/metadata/workflow' -H 'Content-Type: application/json' -d @ai_route_refund_request.json
curl -X POST 'http://localhost:8080/api/metadata/workflow' -H 'Content-Type: application/json' -d @ai_route_sales_lead.json
curl -X POST 'http://localhost:8080/api/metadata/workflow' -H 'Content-Type: application/json' -d @ai_workflow_router.json
```

启动路由器：

```shell
curl -X POST 'http://localhost:8080/api/workflow/ai_workflow_router' \
  -H 'Content-Type: application/json' \
  -d '{"request":"I was charged twice for an order I returned."}'
```

路由器会在输出中记录所选工作流、模型的路由原因以及子工作流 ID。`SUB_WORKFLOW` 会等待所选子工作流完成；子工作流输出可在 `${run_selected_workflow.output}` 获取。

## 安全地扩展目录

要添加一条路由，请同时更新两处：

1. 将工作流名称和描述添加到 LLM 的目录中。
2. 注册一个名称与目录条目完全匹配的工作流的 `1` 版本。

子工作流名称在运行时从 LLM 输出中解析。元数据注册表中不存在的名称无法启动子工作流。

## 相关配方

- [AI Cookbook](../ai/cookbook/index.md) — 聊天、RAG、MCP 智能体和原生 AI 任务的生产级入门示例。
- [以代码编写的动态工作流](dynamic-workflows.md) — 当图本身需要动态生成时，用 Python 构建工作流定义。
