---
description: 验证工作流模式、运行模拟工作流测试，并验证真实的 Conductor 执行。
---

# 验证和测试工作流

使用三层。模式验证捕获无效定义，模拟工作流测试检查编排决策，真实执行验证工作者和集成。

## 前置条件

- 可访问的 Conductor 服务器。
- 保存为 `workflow.json` 的工作流定义。
- 真实工作者和外部依赖仅用于最后的执行层。

## 1. 验证定义

验证检查元数据和图规则，但并不能证明有工作者在轮询或外部端点可达。

```bash
curl -i -X POST 'http://localhost:8080/api/metadata/workflow/validate' \
  -H 'Content-Type: application/json' \
  --data-binary @workflow.json
```

成功是空的 `200 OK` 响应。在注册之前修复验证错误。

## 2. 用模拟任务测试编排

`POST /api/workflow/test` 执行决策逻辑，任务输出按引用名称提供。每个引用映射到一个列表，因为循环或重试可能消耗多个模拟。

```json
--8<-- "docs/devguide/cookbook/examples/workflow-test.json"
```

```bash
curl -sS -X POST 'http://localhost:8080/api/workflow/test' \
  -H 'Content-Type: application/json' \
  --data-binary @workflow-test.json
```

成功是一次任务状态和工作流输出都与预期分支匹配的模拟执行。模拟上的 `executionTime` 和 `queueWaitTime` 可以演练超时行为。嵌套的 `SUB_WORKFLOW` 测试使用 `subWorkflowTestRequest`。

## 3. 运行真实的边界

注册定义，启动它，并检查返回的工作流 ID。

```bash
conductor workflow create workflow.json
conductor workflow start -w order_workflow -i '{"orderId":"order-123"}'
conductor workflow get-execution <workflow-id> -c
```

成功是预期的终态和已验证的任务输出。没有已注册任务定义和轮询工作者的 `SIMPLE` 任务会一直停留在队列中；模拟测试无法检测该部署缺口。

## 限制

模拟测试不会调用工作者、broker、数据库或 HTTP 端点，也无法验证它们的认证、延迟或重试行为。为每个生产边界保留真实的集成测试或冒烟测试。

接下来，使用[可靠性与错误处理](handling-errors.md)添加可靠性策略，并使用[调试与恢复](debugging-workflows.md)演练恢复。
