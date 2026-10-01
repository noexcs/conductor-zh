---
description: "Conductor 工作流定义的完整参考——属性、任务配置、输入表达式、失败工作流与超时策略。"
---

# 工作流定义

工作流定义包含了定义工作流行为所需的全部信息。该定义最重要的部分是 `tasks` 属性，它是一个由 [**任务配置**](#task-configurations) 组成的数组。

关于工作流结构与任务结构的正式 JSON Schema 定义，请参阅 [Schemas](../schemas.md)。所链接的源 Schema 即字段级契约。


## 工作流属性
| 字段                         | 类型                             | 描述                                                                                                                                                                  | 说明                                                                                             |
|:------------------------------|:---------------------------------|:----------------------------------------------------------------------------------------------------------------------------------------------------------------------| :------------------------------------------------------------------------------------------------ |
| name                          | string                           | 工作流名称                                                                                                                                                             |                                                                                                   |
| description                   | string                           | 工作流的描述                                                                                                                                                           | 可选                                                                                          |
| version                       | number                           | 用于标识定义版本的数字字段。请使用递增的数字。                                                                                                                      | 启动工作流执行时，若未指定，则使用版本号最高的定义                                             |
| tasks                         | array of object(s)               | 任务配置数组。[详情](#task-configurations)                                                                                                                           |                                                                                                   |
| inputParameters               | array of string(s)               | 输入参数列表。用于记录工作流所需的输入                                                                                                                                | 可选。                                                                                         |
| outputParameters              | object                           | 用于生成工作流输出的 JSON 模板                                                                                                                                         | 若未指定，输出被定义为*最后一个*执行任务的输出                                                 |
| inputTemplate                 | object                           | 默认输入值。见 [使用 inputTemplate 设置默认输入](#default-input-with-inputtemplate)                                                                                 | 可选。                                                                                         |
| failureWorkflow               | string                           | 当前工作流失败时要运行的工作流。适用于失败时的清理或后续操作。[说明](#failure-workflow)                                                                              | 可选。                                                                                         |
| failureWorkflowVersion        | number                           | 当指定了 `failureWorkflow` 参数时，设置当前工作流失败时要运行的*失败工作流版本*。若未指定，将使用最新版本。                                                         | 可选。                                                                                         |
| schemaVersion                 | number                           | 当前 Conductor Schema 版本。schemaVersion 1 已停止支持。                                                                                                             | 必须为 2                                                                                         |
| restartable                   | boolean                          | 允许重启工作流的标志                                                                                                                                                   | 默认 true                                                                                      |
| workflowStatusListenerEnabled | boolean                          | 启用状态回调。[说明](#workflow-status-listener)                                                                                                                       | 默认 false                                                                                     |
| ownerEmail                    | string                           | 负责该工作流的团队邮箱地址                                                                                                                                             | 必填                                                                                          |
| timeoutSeconds                | number                           | 若工作流未迁移到终态，超过该秒数超时后将被标记为 `TIMED_OUT`                                                                                                         | 设为 0 则不启用超时                                                                           |
| timeoutPolicy                 | string ([enum](#timeout-policy)) | 工作流的超时策略                                                                                                                                                       | 默认 `TIME_OUT_WF`                                                                         |

### 失败工作流 { #failure-workflow }

失败工作流会获得*原失败工作流的输入*，以及 3 项附加内容，

* `workflowId` - 触发失败工作流的那个失败工作流的 ID。
* `reason` - 包含工作流失败原因的字符串。
* `failureStatus` - 失败工作流的状态字符串表示。
* `failureTaskId` - 触发失败工作流的那个工作流中失败任务的 ID。

### 超时策略 { #timeout-policy }

* TIME_OUT_WF: 工作流被标记为 TIMED_OUT 并终止
* ALERT_ONLY: 注册一个计数器（workflow_failure，状态标签设置为 `TIMED_OUT`）

### 工作流状态监听器 { #workflow-status-listener }
将工作流定义中的 `workflowStatusListenerEnabled` 字段设置为 `true` 即可启用通知。

要添加自定义的工作流状态监听器实现，请参阅 [工作流状态监听器扩展指南](../../advanced/extend.md#workflow-status-listener)。

监听器可以这样实现：要么向外部系统发送通知，要么如 [事件处理器指南](../eventhandlers.md) 中所述，在 conductor 队列上发送事件，以完成或使另一个工作流中的另一个任务失败。

### 使用 `inputTemplate` 设置默认输入 { #default-input-with-inputtemplate }

* `inputTemplate` 允许你定义默认输入值，这些值可在运行时（工作流被调用时）选择性地覆盖。
* 例如：在你的工作流定义中，可以这样定义 inputTemplate：

```json
"inputTemplate": {
    "url": "https://some_url:7004"
}
```

如果没有向工作流提供 `url` 作为输入，则 `url` 将为 `https://some_url:7004`。




## 任务配置 { #task-configurations }

工作流定义中的 `tasks` 属性定义了一个*任务配置*数组。这是工作流的蓝图。任务配置可以引用不同类型的任务。

* 简单任务
* 系统任务
* 运算符

注意：任务配置不应与**任务定义**混淆，后者用于注册 SIMPLE（基于工作者）任务。

| 字段             | 类型    | 描述                                                                                                                                    | 说明                                                                 |
| :---------------- | :------ | :--------------------------------------------------------------------------------------------------------------------------------------- | :-------------------------------------------------------------------- |
| name              | string  | 任务名称。在启动工作流之前，必须先作为任务类型在 Conductor 中注册                                                                      |                                                                       |
| taskReferenceName | string  | 用于在工作流内引用该任务的别名。必须在工作流内唯一。                                                                                   |                                                                       |
| type              | string  | 任务类型。由远程工作者执行的任务为 SIMPLE，或其中一种系统任务类型                                                                       |                                                                       |
| description       | string  | 任务的描述                                                                                                                              | 可选                                                                  |
| optional          | boolean | true 或 false。设为 true 时——即使任务失败，工作流也会继续。任务状态反映为 `COMPLETED_WITH_ERRORS`                                        | 默认 `false`                                                   |
| inputParameters   | object  | 定义给任务输入的 JSON 模板。任务中只能使用 `inputParameters` 或 `inputExpression` 之一。                                                | 详情见 [使用表达式](#using-expressions) |
| inputExpression   | object  | 定义给任务输入的 JSONPath 表达式。任务中只能使用 `inputParameters` 或 `inputExpression` 之一。                                          | 详情见 [使用表达式](#using-expressions) |
| asyncComplete     | boolean | `false` 表示执行后标记状态为 COMPLETED；`true` 表示任务保持 IN_PROGRESS，并等待外部事件将其完成。                                       | 默认 `false`                                                   |
| startDelay        | number  | 任务可被工作者轮询之前等待的秒数。                                                                                                       | 默认 0。                                                        |


除这些参数外，系统任务还有自己的参数。更多信息请查阅 [系统任务](systemtasks/index.md)。

### 使用表达式 { #using-expressions }
每个执行的任务都会根据任务配置中配置的 `inputParameters` 模板或 `inputExpression` 获得输入。任务中只能使用 `inputParameters` 或 `inputExpression` 之一。

#### inputParameters
`inputParameters` 可以使用 JSONPath **表达式**从工作流输入和工作流中的其他任务中提取值。

例如，当触发新执行时，客户端/调用方会为工作流提供一个 `input`。工作流的 `input` 可通过形如 `${workflow.input...}` 的*表达式*获取。同样，先前已执行任务的 `input` 和 `output` 数据也可以通过*表达式*提取，供后续任务的 `inputParameters` 使用。

一般来说，`inputParameters` 可以使用以下语法的*表达式*：

> `${SOURCE.input/output.JSONPath}`

| 字段        | 描述                                                              |
| ------------ | ------------------------------------------------------------------------ |
| SOURCE       | 可以是 `"workflow"`，或任意任务的引用名称             |
| input/output | 指来源的输入或输出                       |
| JSONPath     | 用于从来源的输入/输出中提取 JSON 片段的 JSON 路径表达式 |


!!! note "JSON Path 支持"
    Conductor 支持 [JSONPath](http://goessner.net/articles/JsonPath/) 规范，并使用 [jayway/JsonPath](https://github.com/jayway/JsonPath) Java 实现。

!!! note "转义表达式"
    要转义一个表达式，请在其前面添加一个额外的 _$_ 字符（例如：```$${workflow.input...}```）。

#### inputExpression

`inputExpression` 可用于从工作流输入中选择整个对象，或选择另一个任务的输出。该字段支持所有[确定性（definite）](https://github.com/json-path/JsonPath#what-is-returned-when) JSONPath 表达式。

`inputExpression` 中映射值的语法遵循以下模式，

> `SOURCE.input/output.JSONPath`

**注意：**```inputExpression``` 字段不要求用 `${}` 包裹表达式。

参见下文[示例](#example-3-inputexpression)。

## 示例

### 示例 1 - 基础工作流定义 

假设你的业务逻辑是简单地获取一些货运信息，然后执行发货。你可以从逻辑上将其划分为两个任务：

 1. *shipping_info* - 第一个任务接收所提供的账户号，并输出一个地址。  
 2. *shipping_task* - 第二个任务接收地址信息并生成货运标签。

我们可以将这两个任务配置到工作流定义的 `tasks` 数组中。假设 ```shipping info``` 接收一个账户号，并返回姓名和地址。

```json
{
  "name": "mail_a_box",
  "description": "shipping Workflow",
  "version": 1,
  "tasks": [
    {
      "name": "shipping_info",
      "taskReferenceName": "shipping_info_ref",
      "inputParameters": {
        "account": "${workflow.input.accountNumber}"
      },
      "type": "SIMPLE"
    },
    {
      "name": "shipping_task",
      "taskReferenceName": "shipping_task_ref",
      "inputParameters": {
        "name": "${shipping_info_ref.output.name}",
		"streetAddress": "${shipping_info_ref.output.streetAddress}",
		"city": "${shipping_info_ref.output.city}",
		"state": "${shipping_info_ref.output.state}",
		"zipcode": "${shipping_info_ref.output.zipcode}",
      },
      "type": "SIMPLE"
    }
  ],
  "outputParameters": {
    "trackingNumber": "${shipping_task_ref.output.trackingNumber}"
  },
  "failureWorkflow": "shipping_issues",
  "failureWorkflowVersion": 1,
  "restartable": true,
  "workflowStatusListenerEnabled": true,
  "ownerEmail": "conductor@example.com",
  "timeoutPolicy": "ALERT_ONLY",
  "timeoutSeconds": 0,
  "variables": {},
  "inputTemplate": {}
}
```

2 个任务完成后，工作流输出第二个任务生成的跟踪号。如果工作流失败，则运行第二个名为 ```shipping_issues``` 的工作流。


### 示例 2 - 任务配置
考虑任务 `http_task`，其输入配置为使用工作流以及名为 `loc_task` 的任务的输入/输出参数。

```json
{
  "name": "encode_workflow",
  "description": "Encode movie.",
  "version": 1,
  "inputParameters": [
    "movieId", "fileLocation", "recipe"
  ],
  "tasks": [
    {
      "name": "loc_task",
      "taskReferenceName": "loc_task_ref",
      "taskType": "SIMPLE",
      ...      
    },    
    {
      "name": "http_task",
      "taskReferenceName": "http_task_ref",
      "taskType": "HTTP",
      "inputParameters": {
        "movieId": "${workflow.input.movieId}",
        "url": "${workflow.input.fileLocation}",
        "lang": "${loc_task.output.languages[0]}",
        "http_request": {
          "method": "POST",
          "url": "http://example.com/${loc_task.output.fileId}/encode",
          "body": {
            "recipe": "${workflow.input.recipe}",
            "params": {
              "width": 100,
              "height": 100
            }
          },
          "headers": {
            "Accept": "application/json",
            "Content-Type": "application/json"
          }
        }
      }
    }
  ],
  "ownerEmail": "conductor@example.com",
  "variables": {},
  "inputTemplate": {}
}

```

将以下内容视为*工作流输入*

```json
{
  "movieId": "movie_123",
  "fileLocation":"s3://moviebucket/file123",
  "recipe":"png"
}
```
*loc_task* 的输出如下：

```json
{
  "fileId": "file_xxx_yyy_zzz",
  "languages": ["en","ja","es"]
}
```

调度任务时，Conductor 会合并工作流输入与 `loc_task` 输出的值，并为 `http_task` 创建如下输入：

```json
{
  "movieId": "movie_123",
  "url": "s3://moviebucket/file123",
  "lang": "en",
  "http_request": {
    "method": "POST",
    "url": "http://example.com/file_xxx_yyy_zzz/encode",
    "body": {
      "recipe": "png",
      "params": {
        "width": 100,
        "height": 100
      }
    },
    "headers": {
    	"Accept": "application/json",
    	"Content-Type": "application/json"
    }
  }
}
```

### 示例 3 - inputExpression { #example-3-inputexpression }
给定如下任务配置：
```json
{
  "name": "loc_task",
  "taskReferenceName": "loc_task_ref",
  "taskType": "SIMPLE",
  "inputExpression": {
    "expression": "workflow.input",
    "type": "JSON_PATH"
  }  
}
```

当工作流以如下*工作流输入*被调用时
```json
{
  "movieId": "movie_123",
  "fileLocation":"s3://moviebucket/file123",
  "recipe":"png"
}
```

当任务 `loc_task` 被调度时，整个工作流输入对象将作为任务输入传入：
```json
{
  "movieId": "movie_123",
  "fileLocation":"s3://moviebucket/file123",
  "recipe":"png"
}
```
