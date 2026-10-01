---
description: "Join 任务 — 在 Conductor 工作流中同步并行分支，等待所有分叉任务完成。"
---

# Join
```json
"type" : "JOIN"
```

Join 任务与 [Fork](fork-task.md) 或 [Dynamic Fork](dynamic-fork-task.md) 任务配合使用，用于等待并汇聚各分叉。Join 任务还会聚合分叉任务的输出以供后续使用。

Join 任务的行为取决于前面的分叉类型：

* 与静态 Fork 任务配合使用时，Join 任务会等待所提供的分叉任务列表全部完成后再进入下一个任务。
* 与动态 Fork 任务配合使用时，它会隐式地等待所有分叉任务完成。


## 任务参数

与静态 Fork 配合使用时，在 Join 任务配置顶层使用以下参数。

| 参数          | 类型                | 描述                                       | 必填 / 可选  |
| ------------------ | ------------------- | ------------------------------------------------- | -------------------- |
| joinOn    | List[String] | （仅适用于静态 Fork）Join 任务在继续执行下一个任务之前将等待其完成的任务引用名列表。如果未指定，Join 将不等待任何分叉任务完成就直接进入下一个任务。 | 可选。 |

## JSON 配置

以下是 Join 任务的任务配置。

### 配合静态分叉

```json
{
  "name": "join",
  "taskReferenceName": "join_ref",
  "inputParameters": {},
  "type": "JOIN",
  "joinOn": [
    // List of task reference names that the join should wait for
  ]
}
```

### 配合动态分叉

```json
{
  "name": "join",
  "taskReferenceName": "join_ref",
  "inputParameters": {},
  "type": "JOIN"
}
```

## 输出

Join 任务将返回一个包含所有已完成的分叉任务输出的映射（换句话说，即所有 `joinOn` 任务的输出）。键是被汇聚任务的 taskReferenceName，值是对应的任务输出。

**示例：**

```json
{
  "taskReferenceName": {
    "outputKey": "outputValue"
  },
  "anotherTaskReferenceName": {
    "outputKey": "outputValue"
  },
  "someTaskReferenceName": {
    "outputKey": "outputValue"
  }
}
```


## 示例

以下是使用 Join 任务的一些示例。

### 汇聚所有分叉

在此示例任务配置中，Join 任务将按照 `joinOn` 中的指定，等待任务 `my_task_ref_1` 和 `my_task_ref_2` 完成。

```json
[
  {
    "name": "fork_join",
    "taskReferenceName": "my_fork_join_ref",
    "type": "FORK_JOIN",
    "forkTasks": [
      [
        {
          "name": "my_task",
          "taskReferenceName": "my_task_ref_1",
          "type": "SIMPLE"
        }
      ],
      [
        {
          "name": "my_task",
          "taskReferenceName": "my_task_ref_2",
          "type": "SIMPLE"
        }
      ]
    ]
  },
  {
    "name": "join_task",
    "taskReferenceName": "my_join_task_ref",
    "type": "JOIN",
    "joinOn": [
      "my_task_ref_1",
      "my_task_ref_2"
    ]
  }
]
```


### 忽略一个分叉

在此示例任务配置中，[Fork](fork-task.md) 任务派生三个任务：`email_notification` 任务、`sms_notification` 任务和 `http_notification` 任务。

电子邮件和短信通常是尽力而为的投递系统，而基于 HTTP 的通知可以重试直到成功或最终放弃。因此，在设置通知工作流时，你可能会决定在启动电子邮件和短信通知后继续工作流，但让 `http_notification` 任务继续执行而不阻塞工作流的其余部分。

在这种情况下，你可以如下指定 `joinOn` 任务：

```json
[
  {
    "name": "fork_join",
    "taskReferenceName": "my_fork_join_ref",
    "type": "FORK_JOIN",
    "forkTasks": [
      [
        {
          "name": "email_notification",
          "taskReferenceName": "email_notification_ref",
          "type": "SIMPLE"
        }
      ],
      [
        {
          "name": "sms_notification",
          "taskReferenceName": "sms_notification_ref",
          "type": "SIMPLE"
        }
      ],
      [
        {
          "name": "http_notification",
          "taskReferenceName": "http_notification_ref",
          "type": "SIMPLE"
        }
      ]
    ]
  },
  {
    "name": "notification_join",
    "taskReferenceName": "notification_join_ref",
    "type": "JOIN",
    "joinOn": [
      "email_notification_ref",
      "sms_notification_ref"
    ]
  }
]
```

以下是 `notification_join` 的输出。输出是一个映射，其中键是 `joinOn` 任务的 taskReferenceName，对应的值是这些任务的输出。

```json
{
  "email_notification_ref": {
    "email_sent_at": "2021-11-06T07:37:17+0000",
    "email_sent_to": "test@example.com"
  },
  "sms_notification_ref": {
    "sms_sent_at": "2021-11-06T07:37:17+0129",
    "sms_sent_to": "+1-425-555-0189"
  }
}
```
