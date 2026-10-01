---
description: "隔离组（Isolation Groups）—— 将 Conductor 系统任务的执行隔离到专用队列和线程池中，以获得可预测的性能。"
---
# 隔离组

设想一个 HTTP 任务，其调用的 API 延迟很高，任务队列不断堆积，进而影响了其他低延迟 HTTP 任务的执行。

我们可以使用 `isolationgroupId`（任务定义的一个属性）来隔离此类任务的执行，以获得可预测的性能。

当设置了 isolationGroupId 后，执行器 `SystemTaskWorkerCoordinator` 会为这些任务的执行分配一个隔离的队列和一个隔离的线程池。

如果任务定义中未指定 `isolationgroupId`，则回退到默认行为，即执行器在共享线程池中执行所有任务。 

## 示例

** 任务定义 **
```json
{
  "name": "encode_task",
  "retryCount": 3,

  "timeoutSeconds": 1200,
  "inputKeys": [
    "sourceRequestId",
    "qcElementType"
  ],
  "outputKeys": [
    "state",
    "skipped",
    "result"
  ],
  "timeoutPolicy": "TIME_OUT_WF",
  "retryLogic": "FIXED",
  "retryDelaySeconds": 600,
  "responseTimeoutSeconds": 3600,
  "concurrentExecLimit": 100,
  "rateLimitFrequencyInSeconds": 60,
  "rateLimitPerFrequency": 50,
  "isolationgroupId": "myIsolationGroupId"
}
```
** 工作流定义 **
```json
{
  "name": "encode_and_deploy",
  "description": "Encodes a file and deploys to CDN",
  "version": 1,
  "tasks": [
    {
      "name": "encode",
      "taskReferenceName": "encode",
      "type": "HTTP", 
      "inputParameters": {
        "http_request": {
          "uri": "http://localhost:9200/conductor/_search?size=10",
          "method": "GET"
        }
      }
    }
  ],
  "outputParameters": {
    "cdn_url": "${d1.output.location}"
  },
  "failureWorkflow": "cleanup_encode_resources",
  "restartable": true,
  "workflowStatusListenerEnabled": true,
  "schemaVersion": 2
}
```


- 将 `encode` 放入 `HTTP-myIsolationGroupId` 队列，并为其分配一个新的线程池用于执行。

<b>注意：</b>  要启用此功能，需要将 `workflow.isolated.system.task.enable` 属性设为 `true`，其默认值为 `false`

属性 `workflow.isolated.system.task.worker.thread.count`  设置隔离任务的线程池大小；默认值为 `1`。

目前 isolationGroupId 仅在 HTTP 和 kafka 任务中受支持。 

### 执行命名空间（Execution Name Space）

`executionNameSpace` 是 taskdef 的一个属性，可用于为任务执行提供 JVM 级别的隔离，并水平扩展执行器部署。

使用 isolationGroupId 的局限在于：执行器会为每个 `isolationgroupId` 分配一个新的线程池，因此我们需要纵向扩展执行器。  此外，由于执行器在同一个 JVM 中运行任务，任务执行并非完全隔离。 

为了支持 JVM 隔离，同时允许执行器水平扩展，我们可以在 taskdef 中使用 `executionNameSpace` 属性。

执行器会消费其 executionNameSpace 与配置属性 `workflow.system.task.worker.executionNameSpace` 匹配的任务。

如果未设置该属性，执行器将执行未设置任何 executionNameSpace 的任务。 


```json
{
  "name": "encode_task",
  "retryCount": 3,

  "timeoutSeconds": 1200,
  "inputKeys": [
    "sourceRequestId",
    "qcElementType"
  ],
  "outputKeys": [
    "state",
    "skipped",
    "result"
  ],
  "timeoutPolicy": "TIME_OUT_WF",
  "retryLogic": "FIXED",
  "retryDelaySeconds": 600,
  "responseTimeoutSeconds": 3600,
  "concurrentExecLimit": 100,
  "rateLimitFrequencyInSeconds": 60,
  "rateLimitPerFrequency": 50,
  "executionNameSpace": "myExecutionNameSpace"
}
```

#### 示例工作流任务

```json
{ 
  "name": "encode_and_deploy",
  "description": "Encodes a file and deploys to CDN",
  "version": 1,
  "tasks": [
    { 
      "name": "encode",
      "taskReferenceName": "encode",
      "type": "HTTP", 
      "inputParameters": {
        "http_request": {
          "uri": "http://localhost:9200/conductor/_search?size=10",
          "method": "GET"
        }
      }
    }
  ],
  "outputParameters": {
    "cdn_url": "${d1.output.location}"
  },
  "failureWorkflow": "cleanup_encode_resources",
  "restartable": true,
  "workflowStatusListenerEnabled": true,
  "schemaVersion": 2
}
``` 

- `encode` 任务将由其 `workflow.system.task.worker.executionNameSpace` 属性为 `myExecutionNameSpace` 的执行器部署来执行

`executionNameSpace` 可以与 `isolationGroupId` 一起使用。

如果上述任务包含 isolationGroupId `myIsolationGroupId`，任务将被调度到队列 HTTP@myExecutionNameSpace-myIsolationGroupId，并在带有 myExecutionNameSpace 的部署组中获得一个新的线程池用于执行
