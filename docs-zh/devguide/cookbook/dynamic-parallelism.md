---
description: "Conductor 配方——动态并行配方，包括按分支执行不同任务的 Dynamic Fork、同一任务扇出以及并行子工作流。"
---

# 动态并行

### 并行运行不同任务（Dynamic Fork）

当每个并行分支运行**不同**的任务时，使用 `dynamicForkTasksParam` + `dynamicForkTasksInputParamName`。任务列表由前置步骤在运行时确定。

```json
{
  "name": "dynamic_fork_different_tasks",
  "version": 1,
  "schemaVersion": 2,
  "tasks": [
    {
      "name": "prepare_tasks",
      "taskReferenceName": "prepare",
      "type": "INLINE",
      "inputParameters": {
        "evaluatorType": "graaljs",
        "expression": "(function() { return { dynamicTasks: [{name: 'HTTP', taskReferenceName: 'fetch_weather', type: 'HTTP'}, {name: 'HTTP', taskReferenceName: 'fetch_news', type: 'HTTP'}], dynamicTasksInput: { fetch_weather: { http_request: {uri: 'https://api.weather.gov/points/39.7456,-104.9994', method: 'GET'}}, fetch_news: { http_request: {uri: 'https://hacker-news.firebaseio.com/v0/topstories.json', method: 'GET'}}}}; })()"
      }
    },
    {
      "name": "fork_join_dynamic",
      "taskReferenceName": "dynamic_fork",
      "type": "FORK_JOIN_DYNAMIC",
      "inputParameters": {
        "dynamicTasks": "${prepare.output.result.dynamicTasks}",
        "dynamicTasksInput": "${prepare.output.result.dynamicTasksInput}"
      },
      "dynamicForkTasksParam": "dynamicTasks",
      "dynamicForkTasksInputParamName": "dynamicTasksInput"
    },
    {
      "name": "join",
      "taskReferenceName": "join_ref",
      "type": "JOIN"
    }
  ]
}
```

`dynamicTasks` 是任务定义的数组（每个包含 `name`、`taskReferenceName` 和 `type`）。`dynamicTasksInput` 是以每个任务的 `taskReferenceName` 为键的映射，包含其输入负载。

**注册并运行：**

```shell
curl -X POST 'http://localhost:8080/api/metadata/workflow' \
  -H 'Content-Type: application/json' \
  -d @dynamic_fork_different_tasks.json

curl -X POST 'http://localhost:8080/api/workflow/dynamic_fork_different_tasks' \
  -H 'Content-Type: application/json' \
  -d '{}'
```

---

### 并行运行同一任务（扇出）

在多个输入上运行**相同**的任务类型时，使用 `forkTaskName` + `forkTaskInputs`。

```json
{
  "name": "fan_out_http_calls",
  "version": 1,
  "schemaVersion": 2,
  "tasks": [
    {
      "name": "fork_join_dynamic",
      "taskReferenceName": "parallel_fetch",
      "type": "FORK_JOIN_DYNAMIC",
      "inputParameters": {
        "forkTaskName": "HTTP",
        "forkTaskInputs": [
          {"http_request": {"uri": "https://jsonplaceholder.typicode.com/posts/1", "method": "GET"}},
          {"http_request": {"uri": "https://jsonplaceholder.typicode.com/posts/2", "method": "GET"}},
          {"http_request": {"uri": "https://jsonplaceholder.typicode.com/posts/3", "method": "GET"}}
        ]
      }
    },
    {
      "name": "join",
      "taskReferenceName": "join_ref",
      "type": "JOIN"
    }
  ]
}
```

!!! tip
    Conductor 会向每个分叉的输入中注入 `__index`，以便你在结果中追踪每个并行分支的位置。

**注册并运行：**

```shell
curl -X POST 'http://localhost:8080/api/metadata/workflow' \
  -H 'Content-Type: application/json' \
  -d @fan_out_http_calls.json

curl -X POST 'http://localhost:8080/api/workflow/fan_out_http_calls' \
  -H 'Content-Type: application/json' \
  -d '{}'
```

---

### 并行运行子工作流

使用 `forkTaskWorkflow` + `forkTaskInputs` 向另一个工作流的多个实例扇出。

```json
{
  "name": "parallel_sub_workflows",
  "version": 1,
  "schemaVersion": 2,
  "tasks": [
    {
      "name": "fork_join_dynamic",
      "taskReferenceName": "parallel_regions",
      "type": "FORK_JOIN_DYNAMIC",
      "inputParameters": {
        "forkTaskWorkflow": "process_region",
        "forkTaskWorkflowVersion": 1,
        "forkTaskInputs": [
          {"region": "us-east-1", "data": "batch_a"},
          {"region": "eu-west-1", "data": "batch_b"},
          {"region": "ap-southeast-1", "data": "batch_c"}
        ]
      }
    },
    {
      "name": "join",
      "taskReferenceName": "join_ref",
      "type": "JOIN"
    }
  ]
}
```

`forkTaskInputs` 中的每个元素都会生成一个 `process_region` 工作流的实例。所有结果在 JOIN 任务中收集。

**注册并运行：**

```shell
curl -X POST 'http://localhost:8080/api/metadata/workflow' \
  -H 'Content-Type: application/json' \
  -d @parallel_sub_workflows.json

curl -X POST 'http://localhost:8080/api/workflow/parallel_sub_workflows' \
  -H 'Content-Type: application/json' \
  -d '{}'
```
