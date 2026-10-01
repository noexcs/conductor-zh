---
description: 使用 CLI、REST API、Java、Python、TypeScript 或 Go 启动 Conductor 工作流执行。
---

# 启动工作流

启动工作流会创建一个持久执行并返回工作流 ID。请保留该 ID：它是状态、任务、日志和恢复操作的主键。

## 前置条件

- 工作流定义已注册。
- 每个 `SIMPLE` 任务都有任务定义和正在运行的工作者。
- CLI 或所选 SDK 已配置指向同一服务器。

## 使用 CLI 启动

长时间运行的工作请使用异步启动：

```bash
conductor workflow start -w sample_workflow -i '{"service":"fedex"}'
```

当可重复性和可查找性重要时，固定版本并附加业务关联 ID：

```bash
conductor workflow start -w sample_workflow --version 2 \
  --correlation order-123 -i '{"service":"fedex"}'
```

对于有界的测试，`--sync` 会等待执行结果：

```bash
conductor workflow start -w sample_workflow -i '{"service":"fedex"}' --sync
```

成功的标志是：异步启动返回工作流 ID，同步启动返回具有预期状态的工作流结果。

## 使用 REST 启动

`POST /api/workflow/{name}` 直接接受工作流输入映射，并以文本形式返回工作流 ID。

```bash
curl -sS -X POST 'http://localhost:8080/api/workflow/sample_workflow' \
  -H 'Content-Type: application/json' \
  --data '{"service":"fedex"}'
```

当需要 `version`、`correlationId`、`priority` 或 `taskToDomain` 等字段时，使用带 `StartWorkflowRequest` 的 `POST /api/workflow`。仅当调用方需要同步等待时，使用 `POST /api/workflow/execute/{name}/{version}`。完整的请求和响应参考以[启动工作流 API](../../../documentation/api/startworkflow.md)为准。

## 使用 SDK 启动

以下示例展示客户端配置完成后的启动调用。依赖项和认证配置参见每个选项卡下方链接的 SDK 参考。

=== "Java"

    ```java
    StartWorkflowRequest request = new StartWorkflowRequest();
    request.setName("sample_workflow");
    request.setVersion(2);
    request.setCorrelationId("order-123");
    request.setInput(Map.of("service", "fedex"));

    String workflowId = clients.getWorkflowClient().startWorkflow(request);
    ```

    参见 [Java SDK](../../../documentation/clientsdks/java-sdk.md)。

=== "Python"

    ```python
    from conductor.client.http.models import StartWorkflowRequest

    request = StartWorkflowRequest(
        name="sample_workflow",
        version=2,
        correlation_id="order-123",
        input={"service": "fedex"},
    )
    workflow_id = executor.start_workflow(request)
    ```

    参见 [Python SDK](../../../documentation/clientsdks/python-sdk.md)。

=== "TypeScript"

    ```typescript
    const workflowId = await workflowClient.startWorkflow({
      name: "sample_workflow",
      version: 2,
      correlationId: "order-123",
      input: { service: "fedex" },
    });
    ```

    参见 [JavaScript 和 TypeScript SDK](../../../documentation/clientsdks/js-sdk.md)。

=== "Go"

    ```go
    workflowID, err := workflowExecutor.StartWorkflow(&model.StartWorkflowRequest{
        Name:          "sample_workflow",
        Version:       2,
        CorrelationId: "order-123",
        Input: map[string]string{
            "service": "fedex",
        },
    })
    if err != nil {
        return err
    }
    ```

    参见 [Go SDK](../../../documentation/clientsdks/go-sdk.md)。

## 检查执行

```bash
conductor workflow get-execution <workflow-id> -c
```

确认工作流名称和版本、输入、当前状态以及每个任务的状态。仅提交并不证明工作者或集成已完成。

## 限制

- 同步执行会让客户端一直等待，不适合人工任务、定时器和长时间运行的工作者。
- 省略 `version` 会选择服务器上最新注册的版本；当调用方需要可重复行为时请固定它。
- 关联 ID 有助于查找，但不一定唯一，也不能替代工作流 ID。

接下来，学习如何[查看执行](viewing-workflow-executions.md)或[选择自动触发器](choosing-a-trigger.md)。

<a id="using-conductor-ui"></a>
<a id="using-the-cli"></a>
<a id="using-apis"></a>
<a id="using-sdks"></a>
