---
description: 使用 CLI 从 JSON 注册并运行 Conductor 工作流 —— 无需 SDK 或工作者。
---

# 从 JSON 运行工作流

**结果：** 一个已完成、可检查的两步工作流 —— 无需编写工作者。

**用时：** 大约 3 分钟。

更喜欢用代码编写？请从[你的第一个工作流和工作者](first-worker.md)开始。

## 前提条件

完成[连接到 Conductor](connect.md)并验证连接后再继续。此工作流使用内置的 HTTP 和 JSON 任务，因此不需要模型提供商的 API 密钥。

## 1. 创建工作流

将其保存为 `workflow.json`。它调用一个公开的测试端点，并用两个内置系统任务转换响应，因此不需要工作者进程。

```json
{
  "name": "hello_workflow",
  "description": "Fetch a test response and return a compact summary.",
  "version": 1,
  "schemaVersion": 2,
  "tasks": [
    {
      "name": "fetch_data",
      "taskReferenceName": "fetch_ref",
      "type": "HTTP",
      "inputParameters": {
        "http_request": {
          "uri": "https://orkes-api-tester.orkesconductor.com/api",
          "method": "GET"
        }
      }
    },
    {
      "name": "summarize_response",
      "taskReferenceName": "summary_ref",
      "type": "JSON_JQ_TRANSFORM",
      "inputParameters": {
        "response": "${fetch_ref.output.response.body}",
        "queryExpression": "{host: .response.hostName, randomValue: .response.randomInt, summary: (\"Host \" + .response.hostName + \" responded with random value \" + (.response.randomInt|tostring))}"
      }
    }
  ],
  "outputParameters": {
    "summary": "${summary_ref.output.result.summary}",
    "host": "${summary_ref.output.result.host}",
    "randomValue": "${summary_ref.output.result.randomValue}"
  }
}
```

[HTTP 任务](../documentation/configuration/workflowdef/systemtasks/http-task.md)执行请求。[JSON JQ 转换任务](../documentation/configuration/workflowdef/systemtasks/json-jq-transform-task.md)整形其 JSON 输出。

## 2. 注册、运行并验证

```bash
conductor workflow create workflow.json
conductor workflow start -w hello_workflow --sync
```

同步启动会返回工作流执行。验证其状态为 `COMPLETED`，且其输出包含 `summary`、`host` 和 `randomValue`。在 UI 中打开新的执行，检查已完成的 `fetch_ref` 和 `summary_ref` 任务。

由于测试端点是随机的，预期的输出值会变化，但其形状为：

```json
{
  "summary": "Host … responded with random value …",
  "host": "…",
  "randomValue": 123
}
```

## 恢复

- 如果 CLI 无法连接，请回到[连接到 Conductor](connect.md)并验证 URL 和凭据。
- 如果注册报告定义已存在，请删除本地测试定义或更改其版本，然后再创建。
- 如果 HTTP 任务失败，请检查其响应，并用新的执行重试；服务器必须能访问该公开测试端点。

## 走向生产的下一步

现在你拥有一个已验证的系统任务工作流。要运行自己的业务逻辑，请继续[你的第一个工作流和工作者](first-worker.md)。探索[设计模式](../devguide/cookbook/index.md)获取完整的可运行示例，或使用[最佳实践](../devguide/bestpractices.md)添加契约、工作者、重试、测试、部署和运维。
