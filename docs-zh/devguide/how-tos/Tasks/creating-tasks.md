---
description: "在 Conductor 中创建和更新任务定义，为工作者任务和系统任务配置超时、重试、速率限制和输入模板。"
---

# 创建/更新任务定义

[任务定义](../../../documentation/configuration/taskdef.md) 指定任务的通用实现细节：

- 超时策略
- 重试逻辑
- 速率限制和执行限制
- 输入/输出键
- 输入模板

该定义适用于所有工作流中该任务的所有实例。

你可以使用 Conductor UI、CLI 或 API 为以下场景创建任务定义：

- **工作者任务**：所有工作者任务（`SIMPLE`）必须先作为任务定义注册到 Conductor 服务器，才能在工作流中执行。
- **系统任务**：系统任务不需要任务定义，但你可以创建同名的任务定义来自定义重试、超时和速率限制行为。

## 使用 Conductor UI

使用 UI，你可以以可视化方式创建或更新任务定义。

### 创建任务定义

**创建任务定义：**

1. 在左侧导航中，打开 **Definitions**，选择 **Task**。
2. 选择 **Define task**。
3. 在 **Task** 表单中配置任务，或打开 **Code** 选项卡直接编辑 JSON。完整参数参见[任务定义](../../../documentation/configuration/taskdef.md)。
4. 选择 **Save**。

### 更新任务定义

**更新任务定义：**

1. 在左侧导航中，打开 **Definitions**，选择 **Task**，然后选择要更新的任务定义。
2. 在 **Task** 表单或 **Code** 选项卡中修改任务。完整参数参见[任务定义](../../../documentation/configuration/taskdef.md)。
3. 选择 **Save**。

## 使用 CLI

把任务定义保存为 JSON 文件并运行：

```bash
conductor task create taskdef.json
```

该文件可以包含单个任务定义对象，也可以是它们的数组。要更新已有定义，编辑文件后运行：

```bash
conductor task update taskdef.json
```

完整参数的参考指南参见[任务定义](../../../documentation/configuration/taskdef.md)。

## 使用 API

完整参数的参考指南参见[任务定义](../../../documentation/configuration/taskdef.md)。

### 创建任务定义

你还可以使用 Create Task Definition API（`POST /api/metadata/taskdefs`）创建任务定义。该 API 接受任务定义数组，允许你批量创建。

??? note "使用 cURL 的示例"
    ```shell
    curl 'http://localhost:8080/api/metadata/taskdefs' \
      -H 'accept: */*' \
      -H 'content-type: application/json' \
      --data-raw '[{"name":"sample_task_name_1","description":"This is a sample task for demo","responseTimeoutSeconds":10,"timeoutSeconds":30,"inputKeys":[],"outputKeys":[],"timeoutPolicy":"TIME_OUT_WF","retryCount":3,"retryLogic":"FIXED","retryDelaySeconds":5,"inputTemplate":{},"rateLimitPerFrequency":0,"rateLimitFrequencyInSeconds":1}]'
    ```


### 更新任务定义

你可以使用 Update Task Definition API（`PUT /api/metadata/taskdefs`）更新任务定义。该 API 一次只能更新一个任务定义。

??? note "使用 cURL 的示例"
    ```shell
    curl 'http://localhost:8080/api/metadata/taskdefs' \
      -X 'PUT' \
      -H 'accept: */*' \
      -H 'content-type: application/json' \
      --data-raw '{"name":"sample_task_name_1","description":"This is a sample task for demo","responseTimeoutSeconds":10,"timeoutSeconds":30,"inputKeys":[],"outputKeys":[],"timeoutPolicy":"TIME_OUT_WF","retryCount":3,"retryLogic":"FIXED","retryDelaySeconds":5,"inputTemplate":{},"rateLimitPerFrequency":0,"rateLimitFrequencyInSeconds":1}'
    ```


## 使用 SDK

每个[客户端 SDK](../../../documentation/clientsdks/index.md) 都包含调用相同创建和更新端点的元数据客户端方法。当任务注册应属于你的应用或部署代码，而不是手动步骤时，使用它们。

## 复用任务

一个任务在 Conductor 中定义后，可以被复用无数次：

- **在同一工作流中** — 使用不同的任务引用名复用同一任务。
- **跨工作流** — 任何工作流都可以引用任何已注册的任务定义。

在多租户系统中复用任务时，分配给一个任务的所有工作默认进入同一个队列。如果某个"吵闹的邻居"导致轮询延迟，你可以增加工作者数量，或使用 [task-to-domain](../../../documentation/api/taskdomains.md) 把任务负载路由到不同队列。
