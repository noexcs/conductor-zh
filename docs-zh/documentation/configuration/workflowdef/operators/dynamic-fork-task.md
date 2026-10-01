---
description: "在 Conductor 中配置 Dynamic Fork 任务，以运行在运行时确定的并行分支。支持每个分叉运行不同任务或相同任务类型。"
---

# Dynamic Fork
```json
"type" : "FORK_JOIN_DYNAMIC"
```

Dynamic Fork 任务（`FORK_JOIN_DYNAMIC`）用于并行运行任务，其分叉行为（例如任务类型和分叉数量）在运行时确定。这与 [Fork](fork-task.md) 任务形成对比，后者的分叉行为在工作流创建时即已定义。

与 Fork 任务一样，Dynamic Fork 任务之后必须跟随一个 [Join](join-task.md) 任务，等待分叉任务完成后再进入下一个任务。该 Join 任务会收集每个分叉任务的输出。

与 Fork/Join 任务不同，Dynamic Fork 任务每个分叉只能运行一个任务。如果每个分叉需要多个任务，可以使用子工作流。

运行 Dynamic Fork 任务有两种方式：

- **每个分叉运行不同的任务**—使用 `dynamicForkTasksParam` 和 `dynamicForkTasksInputParamName`。
- **所有分叉运行相同的任务**—对任意任务类型使用 `forkTaskType` 和 `forkTaskInputs`，对 Sub Workflow 任务使用 `forkTaskWorkflow` 和 `forkTaskInputs`。


## 任务参数

在 Dynamic Fork 任务配置顶层使用以下参数。分叉任务的输入载荷应与其期望的输入相对应。例如，如果分叉任务是 HTTP 任务，其输入应包含 `http_request`。

### 每个分叉运行不同的任务

要配置 Dynamic Fork 任务，请在任务配置顶层提供 `dynamicForkTasksParam` 和 `dynamicForkTasksInputParamName`，并在 `inputParameters` 中提供基于 `dynamicForkTasksParam` 和 `dynamicForkTasksInputParamName` 的匹配参数。


| 参数          | 类型                | 描述                                       | 必填 / 可选  |
| ------------------ | ------------------- | ------------------------------------------------- | -------------------- |
| dynamicForkTasksParam          | String | `inputParameters` 中的参数名，其值用于调度任务。例如 "dynamicTasks"。               | 必填。 |
| dynamicTasks | List[Task] | 将在各分叉中执行的任务配置列表（每个分叉一个任务） | 必填。 |
| dynamicForkTasksInputParamName | String | `inputParameters` 中的参数名，其值用于传递每个分叉任务所需的输入参数。例如 "dynamicTasksInput"。     | 必填。 |
| dynamicTasksInput | Map[String, Map[String, Any]] | 每个分叉任务的输入。键是每个分叉的任务引用名，值是将传入对应任务的输入参数。  | 必填。 |

[Join](join-task.md) 任务必须在分叉任务之后运行。添加 Join 任务以完成分叉-汇聚操作。

### 所有分叉运行相同的任务（任意任务类型）

在 Dynamic Fork 任务配置的 `inputParameters` 内使用以下参数，使所有分叉执行任意任务类型（Sub Workflow 任务除外）。

| 参数          | 类型                | 描述                                       | 必填 / 可选  |
| ------------------ | ------------------- | ------------------------------------------------- | -------------------- |
| forkTaskType  | String (enum) | 每个分叉中将要执行的任务类型。例如 "HTTP" 或 "SIMPLE"。                                                                      | 必填。 |
| forkTaskName	 | String | 每个分叉中将要执行的 Worker 任务（`SIMPLE`）的名称。                                                                                                                        | 仅当 `forkTaskType` 为 "SIMPLE" 时必填。 |
| forkTaskInputs  | List[Map[String, Any]] | 每个分叉任务的输入。列表项的数量与执行时动态分叉的分支数量相对应。        | 必填。 |

[Join](join-task.md) 任务必须在分叉任务之后运行。同样配置 Join 任务以完成分叉-汇聚操作。

### 所有分叉运行相同的子工作流

在 Dynamic Fork 任务配置的 `inputParameters` 内使用以下参数，使所有分叉执行 [Sub Workflow](sub-workflow-task.md) 任务。

| 参数          | 类型                | 描述                                       | 必填 / 可选  |
| ------------------ | ------------------- | ------------------------------------------------- | -------------------- |
| forkTaskWorkflow  | String | 每个分叉中将要执行的工作流名称。            | 必填。 |
| forkTaskWorkflowVersion	 | Integer | 要执行的工作流版本。如果未指定，将使用最新版本。                                | 可选。 |
| forkTaskInputs  | List[Map[String, Any]] | 每个分叉任务的输入。列表项的数量与执行时动态分叉的分支数量相对应。        | 必填。 |

[Join](join-task.md) 任务必须在分叉任务之后运行。同样配置 Join 任务以完成分叉-汇聚操作。


## JSON 配置

以下是 Dynamic Fork 任务的任务配置。

### 每个分叉运行不同的任务

```json
{
  "name": "fork_join_dynamic",
  "taskReferenceName": "fork_join_dynamic_ref",
  "inputParameters": {
    "dynamicTasks": [ // name of the tasks to execute
      {
        "name": "http",
        "taskReferenceName": "http_ref",
        "type": "HTTP",
        "inputParameters": {}
      },
      { 
        // another task configuration 
      }

    ],
    "dynamicTasksInput": { // inputs for the tasks
      "taskReferenceName" : {
        "key": "value",
        "key": "value"
      },
      "anotherTaskReferenceName" : {
        "key": "value",
        "key": "value"
      }
    }
  },
  "type": "FORK_JOIN_DYNAMIC",
  "dynamicForkTasksParam": "dynamicTasks", // input parameter key that will hold the task names to execute
  "dynamicForkTasksInputParamName": "dynamicTasksInput" // input parameter key that will hold the input parameters for each task
}
```

### 所有分叉运行相同的任务（任意任务类型）

```json
{
  "name": "fork_join_dynamic",
  "taskReferenceName": "fork_join_dynamic_ref",
  "inputParameters": {
    "forkTaskType": "HTTP",
    "forkTaskInputs": [
      {
        // inputs for the first branch
      },
      {
        // inputs for the second branch
      },
      ...
    ]
  },
  "type": "FORK_JOIN_DYNAMIC"
}
```

### 所有分叉运行相同的子工作流

```json
{
  "name": "fork_join_dynamic",
  "taskReferenceName": "fork_join_dynamic_ref",
  "inputParameters": {
    "forkTaskWorkflow": "someWorkflow",
    "forkTaskWorkflowVersion": 1,
    "forkTaskInputs": [
      {
        // inputs for the first branch
      },
      {
        // inputs for the second branch
      },
      ...
    ]
  },
  "type": "FORK_JOIN_DYNAMIC"
}
```


## 示例

以下是使用 Dynamic Fork 任务的一些示例。

### 运行不同的任务

要使每个分叉运行不同的任务，必须使用 `dynamicForkTasksParam` 和 `dynamicForkTasksInputParamName`。

在此示例工作流中，Dynamic Fork 任务派生三个分叉，每个分叉运行不同的任务（`HTTP`、`SIMPLE` 和 `INLINE`）。为了获得真正的动态性，你可以再添加一个任务，为 Dynamic Fork 任务准备任务列表和输入。

```json
{
  "name": "DynamicForkExample",
  "description": "This workflow runs different tasks in a dynamic fork.",
  "version": 1,
  "tasks": [
    {
      "name": "fork_join_dynamic",
      "taskReferenceName": "fork_join_dynamic_ref",
      "inputParameters": {
        "dynamicTasks": [
          {
            "name": "inline",
            "taskReferenceName": "task1",
            "type": "INLINE",
            "inputParameters": {
              "expression": "(function () {\n  return $.input;\n})();",
              "evaluatorType": "javascript"
            }
          },
          {
            "name": "http",
            "taskReferenceName": "task2",
            "type": "HTTP",
            "inputParameters": {}
          },
          {
            "name": "task_38",
            "taskReferenceName": "simple_ref",
            "type": "SIMPLE"
          }
        ],
        "dynamicTasksInput": {
          "task1": {
            "input": "one"
          },
          "task2": {
            "http_request": {
              "method": "GET",
              "uri": "https://randomuser.me/api/",
              "connectionTimeOut": 3000,
              "readTimeOut": "3000",
              "accept": "application/json",
              "contentType": "application/json",
              "encode": true
            }
          },
          "task3": {
            "input": {
              "someKey": "someValue"
            }
          }
        }
      },
      "type": "FORK_JOIN_DYNAMIC",
      "dynamicForkTasksParam": "dynamicTasks",
      "dynamicForkTasksInputParamName": "dynamicTasksInput"
    },
    {
      "name": "join",
      "taskReferenceName": "join_ref",
      "inputParameters": {},
      "type": "JOIN",
      "joinOn": []
    }
  ],
  "inputParameters": [],
  "outputParameters": {},
  "schemaVersion": 2,
  "ownerEmail": "example@email.com"
}
```

有关 Fork 中 Join 部分的更多细节，请参阅 [Join](join-task.md) 任务。

### 运行相同的任务 — Worker 任务

在此示例工作流中，使用 Dynamic Fork 任务运行 Worker 任务（`SIMPLE`），这些任务将调整上传图片的尺寸，并把调整后的图片存储到指定的 `location`。

当与 `forkTaskType`（或 `forkTaskWorkflow`）一起使用 `forkTaskInputs` 时，`dynamicForkTasksParam` 和 `dynamicForkTasksInputParamName` 字段不是必需的。

```json
{
  "name": "image_multiple_convert_resize_fork",
  "description": "Image multiple convert resize example",
  "version": 1,
  "tasks": [
    {
      "name": "image_multiple_convert_resize_dynamic_task",
      "taskReferenceName": "image_multiple_convert_resize_dynamic_task_ref",
      "inputParameters": {
        "forkTaskName": "fork_task",
        "forkTaskType": "SIMPLE",
        "forkTaskInputs": [
           {
            "image" : "url1",
            "location" : "location_url",
            "width" : 100,
            "height" : 200
           },
           {
            "image" : "url2",
            "location" : "location_url",
            "width" : 300,
            "height" : 400
           }
       ]
      },
      "type": "FORK_JOIN_DYNAMIC"
    },
    {
      "name": "image_multiple_convert_resize_join",
      "taskReferenceName": "image_multiple_convert_resize_join_ref",
      "inputParameters": {},
      "type": "JOIN"
    }
  ],
  "inputParameters": [],
  "outputParameters": {
    "output": "${join_task_ref.output}"
  },
  "schemaVersion": 2,
  "ownerEmail": "example@email.com"
}
```

有关 Fork 中 Join 部分的更多细节，请参阅 [Join](join-task.md) 任务。


### 运行相同的任务 — HTTP 任务

在此示例工作流中，Dynamic Fork 任务并行运行 HTTP 任务。`forkTaskInputs` 中提供的输入包含 HTTP 任务所期望的典型载荷。

```json
{
  "name": "dynamic_workflow_array_http",
  "description": "Dynamic workflow array - run HTTP tasks",
  "version": 1,
  "tasks": [
    {
      "name": "dynamic_workflow_array_http",
      "taskReferenceName": "dynamic_workflow_array_http_ref",
      "inputParameters": {
        "forkTaskType": "HTTP",
        "forkTaskInputs": [
          {
            "http_request": {
              "method": "GET",
              "uri": "https://randomuser.me/api/"
            }
          },
          {
            "http_request": {
              "method": "GET",
              "uri": "https://randomuser.me/api/"
            }
          }
        ]
      },
      "type": "FORK_JOIN_DYNAMIC"
    },
    {
      "name": "dynamic_workflow_array_http_join",
      "taskReferenceName": "dynamic_workflow_array_http_join_ref",
      "inputParameters": {},
      "type": "JOIN",
      "joinOn": []
    }
  ],
  "inputParameters": [],
  "outputParameters": {},
  "schemaVersion": 2,
  "ownerEmail": "example@email.com"
}
```

有关 Fork 中 Join 部分的更多细节，请参阅 [Join](join-task.md) 任务。


### 运行相同的任务 — 简化配置

使用 `forkTaskInputs` 时，可以使用不含 `dynamicForkTasksParam` 和 `dynamicForkTasksInputParamName` 的简化配置。此方法使用 `forkTaskName` 直接指定任务类型。

```json
{
  "name": "dynamic_fork_simple",
  "description": "Dynamic fork with simplified configuration",
  "version": 1,
  "tasks": [
    {
      "name": "dynamic_fork_http",
      "taskReferenceName": "dynamic_fork_http_ref",
      "inputParameters": {
        "forkTaskName": "HTTP",
        "forkTaskInputs": [
          {
            "uri": "https://orkes-api-tester.orkesconductor.com/api",
            "method": "GET",
            "accept": "application/json",
            "contentType": "application/json",
            "encode": true
          },
          {
            "uri": "https://orkes-api-tester.orkesconductor.com/api",
            "method": "GET",
            "accept": "application/json",
            "contentType": "application/json",
            "encode": true
          }
        ]
      },
      "type": "FORK_JOIN_DYNAMIC"
    },
    {
      "name": "dynamic_fork_http_join",
      "taskReferenceName": "dynamic_fork_http_join_ref",
      "inputParameters": {},
      "type": "JOIN"
    }
  ],
  "inputParameters": [],
  "outputParameters": {},
  "schemaVersion": 2,
  "ownerEmail": "example@email.com"
}
```

有关 Fork 中 Join 部分的更多细节，请参阅 [Join](join-task.md) 任务。


### 运行相同的任务 — Sub Workflow 任务


在此示例工作流中，动态分叉并行运行 Sub Workflow 任务。每个子工作流将调整图片尺寸，并把调整后的图片存储到指定的 `location`。

```json
{
  "name": "image_multiple_convert_resize_fork_subwf",
  "description": "Image multiple convert resize example",
  "version": 1,
  "tasks": [
    {
      "name": "image_multiple_convert_resize_dynamic_task_subworkflow",
      "taskReferenceName": "image_multiple_convert_resize_dynamic_task_subworkflow_ref",
      "inputParameters": {
        "forkTaskWorkflow": "image_resize_subworkflow",
        "forkTaskInputs": [
          {
            "image": "url1",
            "location": "location url",
            "width": 100,
            "height": 200
          },
          {
            "image": "url2",
            "location": "locationurl",
            "width": 300,
            "height": 400
          }
        ]
      },
      "type": "FORK_JOIN_DYNAMIC"
    },
    {
      "name": "dynamic_workflow_array_http_subworkflow",
      "taskReferenceName": "dynamic_workflow_array_http_subworkflow_ref",
      "inputParameters": {},
      "type": "JOIN",
      "joinOn": []
    }
  ],
  "inputParameters": [],
  "outputParameters": {},
  "schemaVersion": 2,
  "ownerEmail": "example@email.com"
}
```

有关 Fork 中 Join 部分的更多细节，请参阅 [Join](join-task.md) 任务。
