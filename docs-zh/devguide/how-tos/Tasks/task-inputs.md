---
description: "在 Conductor 工作流中配置任务输入——在这个开源工作流编排引擎中，使用动态表达式引用工作流输入、任务输出和变量。"
---

# 配置任务输入

在 Conductor 中，任务输入可以在工作流定义中通过多种方式提供：

- 作为硬编码的值 – 
```
"taskInputA": true
```
- 作为对工作流输入、工作流变量或先前任务输入/输出的动态引用 – 
```
"taskInputA": "${workflow.input.someValue}
```

## 动态引用的语法

所有动态引用都采用以下表达式格式：

```
"${type.jsonpath}"
```

这些动态引用采用点号表示法的表达式格式，沿用 [JSONPath 语法](https://goessner.net/articles/JsonPath/)。

| 组成部分            | 描述                                                                                                                    |
| -------------------- | ----------------------------------------------------------------------------------------------------- |
| `${...}`             | 根标记，表示该变量将在运行时被动态替换。             |
| type                 | 引用的类型。支持的取值：<ul><li>**workflow**—指当前工作流实例。</li> <li>**workflow.input**—指工作流的输入参数。</li> <li>**workflow.output**—指工作流的输出参数。</li> <li>**workflow.variables**—指使用 [Set Variable](../../../documentation/configuration/workflowdef/operators/set-variable-task.md) 任务在工作流中设置的工作流变量。</li> <li>**_taskReferenceName_**—按引用名指当前工作流实例中的一个任务。（例如 "http_ref"。）</li> <li>**_taskReferenceName_.input**—指任务的输入参数。</li> <li>**_taskReferenceName_.output**—指任务的输出参数。</li></ul>…
| jsonpath             | 点号表示法的 [JSONPath](https://goessner.net/articles/JsonPath/) 表达式。                 |


### 表达式示例

以下是你可以使用的动态引用（非穷举）列表：

- 引用任务的输入载荷 –        
```
${<taskReferenceName>.input}
```
- 引用任务的输出载荷 –        
```
${<taskReferenceName>.output}
```
- 引用任务的输入参数 –        
```
${<taskReferenceName>.input.<someKey>}
```
- 引用任务的输出参数 –        
```
${<taskReferenceName>.output.<someKey>}
```
- 引用工作流的输入载荷 –        
```
${workflow.input}
```
- 引用工作流的输出载荷 –        
```
${workflow.output}
```
- 引用工作流的输入参数 –        
```
${workflow.input.<someKey>}
```
- 引用工作流的输出参数 –        
```
${workflow.output.<someKey>}
```
- 引用工作流的当前状态（RUNNING、PAUSED、TIMED_OUT、TERMINATED、FAILED 或 COMPLETED） –        
```
${workflow.status}
```
- 引用工作流（执行）ID –        
```
${workflow.workflowId}
```
- （用于子工作流）引用父工作流（执行）ID –        
```
${workflow.parentWorkflowId}
```
- （用于子工作流）引用父工作流中 Sub Workflow 任务的任务执行 ID –        
```
${workflow.parentWorkflowTaskId}
```
- 引用工作流的名称 –        
```
${workflow.workflowType} 
```  
- 引用工作流的版本 –        
```
${workflow.version} 
```    
- 引用工作流执行的开始时间 –        
```
${workflow.createTime} 
```  
- 引用工作流的关联 ID –        
```
${workflow.correlationId} 
```  
- 引用工作流执行期间被调用的域名 –        
```
${workflow.taskToDomain.<domainName>} 
```  
- 引用使用 Set Variable 任务创建的工作流变量 –        
```
${workflow.variables.<someKey>} 
```   


## 示例

以下是工作流中使用动态引用的一些示例。

<details>
<summary>引用工作流输入</summary>

对于给定的工作流输入：

```json
{
  "userID": 1,
  "userName": "SAMPLE",
  "userDetails": {
    "country": "nestedValue",
    "age": 50
  }
}
```

你可以在其他地方使用以下表达式引用这些工作流输入：

```json
{
  "user": "${workflow.input.userName}",
  "userAge": "${workflow.input.userDetails.age}"
}
```

运行时，参数将是：

```json
{
  "user": "SAMPLE",
  "userAge": 50
}
```

</details>

<details>
<summary>引用其他任务输出</summary>

如果任务 <code>previousTaskReference</code> 产生了以下输出：

```json
{
  "taxZone": "A",
  "productDetails": {
    "nestedKey1": "outputValue-1",
    "nestedKey2": "outputValue-2"
  }
}
```

你可以在其他地方使用以下表达式引用这些任务输出：

```json
{
  "nextTaskInput1": "${previousTaskReference.output.taxZone}",
  "nextTaskInput2": "${previousTaskReference.output.productDetails.nestedKey1}"
}
```

运行时，参数将是：

```json
{
  "nextTaskInput1": "A",
  "nextTaskInput2": "outputValue-1"
}
```

</details>

<details>
<summary>引用工作流变量</summary>

如果工作流变量是使用 Set Variable 任务设置的：

```json
{
  "name": "Ipsum"
}
```

该变量可以在同一工作流中使用以下表达式引用：

```json
{
  "user": "${workflow.variables.name}"
}
```

<b>Note:</b> 工作流变量不能跨工作流再次引用，即使在父工作流和子工作流之间也不行。

</details>


<details>
<summary>引用父工作流与子工作流之间的数据</summary>

要把参数从父工作流传递到其子工作流，必须将它们声明为 Sub Workflow 任务的输入参数。如有需要，随后可以使用 Set Variable 任务在子工作流定义内部将这些输入设置为工作流变量。

```
// parent workflow definition with task configuration

{
 "createTime": 1733980872607,
 "updateTime": 0,
 "name": "testParent",
 "description": "workflow with subworkflow",
 "version": 1,
 "tasks": [
   {
     "name": "get_item",
     "taskReferenceName": "get_item_ref",
     "inputParameters": {
       "uri": "https://example.com/api",
       "method": "GET",
       "accept": "application/json",
       "contentType": "application/json",
       "encode": true
     },
     "type": "HTTP",
   },
   {
     "name": "sub_workflow",
     "taskReferenceName": "sub_workflow_ref",
     "inputParameters": {
       "user": "${workflow.variables.name}",
       "item": "${previous_task_ref.output.item[0]}"
     },
     "type": "SUB_WORKFLOW",
     "subWorkflowParam": {
       "name": "testSub",
       "version": 1
     }
   }
 ],
 "inputParameters": [],
 "outputParameters": {}
}
```


要把参数从子工作流传回其父工作流，必须在子工作流定义中把它们作为子工作流的输出参数传递。

```
// sub-workflow definition

{
 "createTime": 1726651838873,
 "updateTime": 1733983507294,
 "name": "testSub",
 "description": "subworkflow for parent workflow",
 "version": 1,
 "tasks": [
   {
     "name": "get-user",
     "taskReferenceName": "get-user_ref",
     "inputParameters": {
       "uri": "https://example.com/api",
       "method": "GET",
       "accept": "application/json",
       "contentType": "application/json",
       "encode": true
     },
     "type": "HTTP",
   },
   {
     "name": "send-notification",
     "taskReferenceName": "send-notification_ref",
     "inputParameters": {
       "uri": "https://example.com/api",
       "method": "GET",
       "accept": "application/json",
       "contentType": "application/json",
       "encode": true
     },
     "type": "HTTP",
   }
 ],
 "inputParameters": [],
 "outputParameters": {
   "location": "${get-user_ref.output.response.body.results[0].location.country}",
   "isNotif": "${send-notification_ref.output}"
 }
}
```

在父工作流中，可以使用表达式格式 `${<sub_workflow_ref>.output.<someKey>}` 引用这些子工作流输出。

</details>


## 故障排查

你可以在 UI 中检查任务执行的输入/输出值，以验证数据是否传递正确。常见错误：

- 如果引用表达式格式不正确，被引用的参数值可能会得到错误的数据或 null 值。
- 如果被引用的值（如任务输出）在被引用的时点尚未解析，被引用的参数值将为 null。
