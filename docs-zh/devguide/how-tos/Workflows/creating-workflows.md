---
description: 使用 CLI、API 或 UI 编写、验证并注册带版本的 Conductor 工作流定义。
---

# 创建或更新工作流

工作流定义是一个带版本的 JSON 文档。它声明工作流的名称、输入和输出，以及它运行的任务。本页介绍如何编写该文档、验证它，并将其注册到服务器。

## 前置条件

- 可访问的 Conductor 服务器和已配置的 CLI。
- 每个 `SIMPLE` 任务都有任务定义和轮询工作者。

## 1. 编写定义

最简定义命名工作流、列出其任务，并映射其输入和输出：

```json
{
  "name": "order_flow",
  "version": 1,
  "schemaVersion": 2,
  "inputParameters": ["orderId"],
  "tasks": [
    {
      "name": "process_order",
      "taskReferenceName": "process_order_ref",
      "type": "SIMPLE",
      "inputParameters": {
        "orderId": "${workflow.input.orderId}"
      }
    }
  ],
  "outputParameters": {
    "status": "${process_order_ref.output.status}"
  }
}
```

将其保存为 `workflow.json`。需要遵循几条规则：

- 为每个任务提供唯一且描述性的 `taskReferenceName`。其他任务通过该名称引用其输出。
- 当某个[内置任务](../Tasks/choosing-tasks.md)能覆盖该操作时，优先使用它。`SIMPLE` 任务需要已注册的任务定义和轮询工作者，否则在运行时一直停留在队列中。
- 保持 `outputParameters` 跨版本稳定，因为调用方依赖它们。

[工作流定义参考](../../../documentation/configuration/workflowdef/index.md)记录了所有字段。

## 2. 注册前验证

```bash
curl -i -X POST 'http://localhost:8080/api/metadata/workflow/validate' \
  -H 'Content-Type: application/json' \
  --data-binary @workflow.json
```

成功是空的 `200 OK` 响应。验证检查的是定义本身，而不是工作者可用性或外部连接性。

## 3. 注册定义

```bash
conductor workflow create workflow.json
```

成功是可通过以下方式看到的已注册名称和版本：

```bash
conductor workflow get <workflow-name>
```

对应的 REST 端点：创建为 `POST /api/metadata/workflow`，更新为 `PUT /api/metadata/workflow`（请求体为定义数组）。两个端点参见[元数据 API](../../../documentation/api/metadata.md)。

## 4. 验证 SIMPLE 任务依赖

列出已注册的任务定义，并与工作流中每个 `type` 为 `SIMPLE` 的任务对比：

```bash
conductor taskDef list
```

然后验证有工作者在轮询每个确切的类型。仅注册不会启动工作者。

## 5. 测试和运行

使用[验证和测试工作流](testing-workflows.md)通过 `/api/workflow/test` 模拟分支，然后针对测试依赖运行一次真实执行。

## 安全地更新和版本管理

<a id="updating-workflows"></a>

当输入、输出、任务顺序或失败语义发生变化且调用方能观察到时，使用新版本。注册新版本，有意识地更新调用方，并在现有调用方或执行仍需要时保留旧版本可用。参见[管理工作流版本](versioning-workflows.md)。

## 在 UI 中创建

<a id="ui-alternative"></a>

1. 在左侧导航中打开**定义**，并选择**工作流**。
2. 在右上角选择**定义工作流**。编辑器打开时显示一个空的开始到结束图。
3. 在**工作流详情**下，输入唯一的名称和描述。
4. 以可视化方式或 JSON 添加任务：
    - 选择画布上的 **+** 节点以插入任务，然后在**任务**面板中配置它。
    - 或打开**代码**选项卡并粘贴完整的 JSON 定义。
5. 选择**保存**。先解决编辑器报告的任何警告。

要修改现有工作流，从**定义**再到**工作流**中打开它，编辑后保存。在自动化流程中使用 CLI/API 流程，使签入的定义保持为唯一可信来源。

## 限制

- 定义验证不会验证任务工作者的部署、凭据、broker 主题或 HTTP 可达性。
- 就地更新同一版本会使发布和回滚更难推理。
- 大的输入/输出负载应放在外部存储中；工作流中只携带引用。

接下来，[启动工作流](starting-workflows.md)并检查返回的执行。

<a id="using-conductor-ui"></a>
<a id="using-the-cli"></a>
<a id="using-apis"></a>
<a id="using-sdks"></a>
