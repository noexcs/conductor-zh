---
description: "Set Variable 任务 — 在 Conductor 中存储和更新工作流级别的变量，供后续任务使用。"
---
# Set Variable

```json
"type" : "SET_VARIABLE"
```

Set Variable 任务（`SET_VARIABLE`）允许你在工作流级别构建跨任务的共享变量。

这些变量可以在工作流的任何位置被初始化、访问或覆盖：

* 一旦初始化，可以在任何后续任务中使用 "${workflow.variables._someName_}" 引用该变量（将 _someName_ 替换为实际的变量名）。
* 已初始化的值可以被后续的 Set Variable 任务覆盖。

## 任务参数

要配置 Set Variable 任务，请在 `inputParameters` 中设置所需的变量名及其对应的值。值可以通过以下两种方式设置：

* 硬编码在工作流定义中；或
* 动态引用。

## JSON 配置

以下是 Set Variable 任务的任务配置。

```json
{
  "name": "set_variable",
  "taskReferenceName": "set_variable_ref",
  "type": "SET_VARIABLE",
  "inputParameters": {
    "variableName": "value",
    "variableName2": "${workflow.input.someKey}"
    "variableName3": 5,
  }
}
```

## 示例

在此示例工作流中，用户名被存储为一个变量，以便在其他需要用户名的任务中复用。

```json
{
  "name": "Welcome_User_Workflow",
  "description": "Designate a user to be welcomed",
  "tasks": [
    {
      "name": "set_name",
      "taskReferenceName": "set_name_ref",
      "type": "SET_VARIABLE",
      "inputParameters": {
        "name": "${workflow.input.userName}"
      }
    },
    {
      "name": "greet_user",
      "taskReferenceName": "greet_user_ref",
      "inputParameters": {
        "var_name": "${workflow.variables.name}"
      },
      "type": "SIMPLE"
    },
    {
      "name": "send_reminder_email",
      "taskReferenceName": "send_reminder_email_ref",
      "inputParameters": {
        "var_name": "${workflow.variables.name}"
      },
      "type": "SIMPLE"
    }
  ]
}
```

在上面的示例中，`set_name` 是一个 Set Variable 任务，它使用工作流输入引用初始化变量 `name`。在后续任务中，该变量通过 "${workflow.variables.name}" 被引用。


## 局限性

使用 Set Variable 任务时存在以下一些局限性：

* **载荷限制**—默认情况下，在 JVM 系统属性（`conductor.max.workflow.variables.payload.threshold.kb`）中定义了变量载荷大小的硬限制为 256KB。超出此限制将导致 Set Variable 任务失败。
* **变量作用域**—Set Variable 任务的作用域仅限于其所在工作流。在一个工作流中初始化的变量不会传递到另一个工作流或子工作流，必须使用另一个 Set Variable 任务重新初始化。
