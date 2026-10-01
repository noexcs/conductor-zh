---
description: "任务域（Task Domains）— 将 Conductor 任务路由到特定的工作者组，用于隔离开发、测试和金丝雀部署。"
---
# 任务域
任务域有助于支持任务开发。其思想是：同一个"任务定义"可以在不同的"域"（domain）中实现。域是开发者控制的任意名称。因此，当工作流启动时，调用方可以指定工作流中的所有任务里哪些需要在特定域中运行，然后在客户端轮询该域来执行这些任务。  

例如，如果工作流（WF1）有 3 个任务 T1、T2、T3。该工作流已部署并运行正常，意味着有 T2 工作者在轮询和执行。如果你修改了 T2 并在本地运行，由于任务来自通用的 T2 队列，无法保证你修改后的 T2 工作者会拿到你想要的那个任务。"任务域"功能通过将 T2 队列按域拆分来解决这个问题，这样当应用以特定域轮询任务 T2 时，就能拿到正确的任务。

启动工作流时可以指定多个域作为回退，例如 "domain1,domain2"。Conductor 会跟踪每个任务的上次轮询时间，因此在这种情况下，它会检查 "domain1" 是否有活跃工作者（在 10 秒窗口内至少轮询过一次的工作者）；如果有，任务就放入 "domain1"；如果没有，则按顺序对下一个域 "domain2" 做同样的检查，依此类推。

如果所提供的域没有活跃工作者：

- 如果 `NO_DOMAIN` 作为域列表中的最后一个 token 提供，则不设置域。
- 否则，任务会被加入域列表中最后一个不活跃的域，希望该域的工作者很快可用。

另外，可以使用 `*` token 为所有任务应用域。通过提供与 `*` 并存的特定任务映射可以覆盖它。 

例如，以下配置：

```json
"taskToDomain": {
  "*": "mydomain",
  "some_task_x":"NO_DOMAIN",
  "some_task_y": "someDomain, NO_DOMAIN",
  "some_task_z": "someInactiveDomain1, someInactiveDomain2"
}
```

- 将 `some_task_x` 放入默认队列（无域）。
- 将 `some_task_y` 放入 `someDomain` 域（如果可用），否则放入默认域。
- 将 `some_task_z` 放入 `someInactiveDomain2`（即使工作者尚不可用）。
- 并将所有其他任务放入 `mydomain`（即使工作者不可用）。


<b>注意</b>：这种"回退"类型的域字符串只能在启动工作流时使用；从客户端轮询时只使用一个域。另外，`NO_DOMAIN` token 应当放在最后。

## 如何使用任务域
### 修改轮询调用
轮询调用现在必须指定域。 

#### Java 客户端
如果你使用的是 Java 客户端，一个简单的属性变更就能让 TaskRunnerConfigurer 将域传递给轮询器。
```
	conductor.worker.T2.domain=mydomain //Task T2 needs to poll for domain "mydomain"
```
#### REST 调用
`GET {{ api_prefix }}/tasks/poll/batch/T2?workerid=myworker&domain=mydomain`
`GET {{ api_prefix }}/tasks/poll/T2?workerid=myworker&domain=mydomain`

### 修改启动工作流调用
启动工作流时，请确保传递了任务到域的映射。

#### Java 客户端
```java
{Map<String, Object> input = new HashMap<>();
input.put("wf_input1", "one");

Map<String, String> taskToDomain = new HashMap<>();
taskToDomain.put("T2", "mydomain");

// Other options ...
// taskToDomain.put("*", "mydomain, NO_DOMAIN")
// taskToDomain.put("T2", "mydomain, fallbackDomain1, fallbackDomain2")

StartWorkflowRequest swr = new StartWorkflowRequest();
swr.withName("myWorkflow")
	.withCorrelationId("corr1")
	.withVersion(1)
	.withInput(input)
	.withTaskToDomain(taskToDomain);

wfclient.startWorkflow(swr);

```

#### REST 调用
`POST {{ api_prefix }}/workflow`

```json
{
  "name": "myWorkflow",
  "version": 1,
  "correlatonId": "corr1"
  "input": {
	"wf_input1": "one"
  },
  "taskToDomain": {
	"*": "mydomain",
	"some_task_x":"NO_DOMAIN",
    "some_task_y": "someDomain, NO_DOMAIN"
  }
}

```
