---
description: "使用类型安全的任务定义和工作流管理，用 Go 构建 Conductor 工作者。"
source_repo: "https://github.com/conductor-oss/go-sdk"
sdk_page: go
---

# Go SDK

## 安装

1. 初始化你的模块。例如：

```shell
mkdir hello_world
cd hello_world
go mod init hello_world
```

2. 获取 SDK：

```shell
go get github.com/conductor-sdk/conductor-go
```

## Hello World

在该仓库中，你可以在 [examples/hello_world](https://github.com/conductor-oss/go-sdk/blob/main/examples/hello_world/) 下找到一个基础的 "Hello World" 示例。

让我们分 3 步来剖析这个应用。


> [!note]
> 你需要一个正在运行的 Conductor Server。
>
> 有关如何运行 Conductor 的详细信息，请参阅[我们的指南](../../devguide/running/deploy.md)。
>
> 示例假定服务器正在监听 http://localhost:8080。


### 步骤 1：通过代码创建工作流 { #step-1-creating-the-workflow-by-code }

"greetings" 工作流将通过代码创建并注册到 Conductor 中。

查看 [examples/hello_world/src/workflow.go](https://github.com/conductor-oss/go-sdk/blob/main/examples/hello_world/src/workflow.go) 中的 `CreateWorkflow` 函数。

```go
func CreateWorkflow(executor *executor.WorkflowExecutor) *workflow.ConductorWorkflow {
	wf := workflow.NewConductorWorkflow(executor).
		Name("greetings").
		Version(1).
		Description("Greetings workflow - Greets a user by their name").
		TimeoutPolicy(workflow.TimeOutWorkflow, 600)

	greet := workflow.NewSimpleTask("greet", "greet_ref").
		Input("person_to_be_greated", "${workflow.input.name}")

	wf.Add(greet)

	wf.OutputParameters(map[string]interface{}{
		"greetings": greet.OutputRef("hello"),
	})

	return wf
}
```

在上面的代码中，我们首先通过调用 `workflow.NewConductorWorkflow(..)` 创建一个工作流，并设置其属性 `Name`、`Version`、`Description` 和 `TimeoutPolicy`。

然后我们创建一个类型为 `"greet"`、引用名为 `"greet_ref"` 的 [Simple Task](https://orkes.io/content/reference-docs/worker-task)，并将其添加到工作流中。该任务以键 `"person_to_be_greated"` 接收工作流输入 `"name"` 作为输入。

> [!note]
> `"person_to_be_greated"` 这个名字太冗长了！为什么要这样命名？
>
> 只是为了说明工作流输入不会被自动传递。
>
> 由于工作流定义中存在 `Input("person_to_be_greated", "${workflow.input.name}")` 这一映射，工作者会获取工作流输入的实际值。
>
> 像 `"${workflow.input.name}"` 这样的表达式会在执行时被其值替换。

最后，通过调用 `wf.OutputParameters(..)` 设置工作流的输出。

`"greetings"` 的值将是已执行的 `"greet"` 任务输出中 `"hello"` 的值，例如：如果任务输出为：
```
{
	"hello" : "Hello, John"
}
```

预期的工作流输出为：
```
{
	"greetings": "Hello, John"
}
```

Go 代码转换为如下 JSON 定义。注册工作流后，你可以在 Conductor 服务器中查看。

```json
{
  "schemaVersion": 2,
  "name": "greetings",
  "description": "Greetings workflow - Greets a user by their name",
  "version": 1,
  "tasks": [
    {
      "name": "greet",
      "taskReferenceName": "greet_ref",
      "type": "SIMPLE",
      "inputParameters": {
        "name": "${workflow.input.name}"
      }
    }
  ],
  "outputParameters": {
    "Greetings": "${greet_ref.output.greetings}"
  },
  "timeoutPolicy": "TIME_OUT_WF",
  "timeoutSeconds": 600
}
```

> [!note]
> 工作流也可以通过 API 注册。使用上述 JSON，你可以发起如下请求：
> ```shell
> curl -X POST -H "Content-Type:application/json" \
> http://localhost:8080/api/metadata/workflow -d @greetings_workflow.json
> ```

在[步骤 3](#step-3-running-the-application)中，你将看到如何创建 `executor.WorkflowExecutor` 实例。


### 步骤 2：创建工作者

工作者是一个用于执行特定任务的函数。

在这个示例中，工作者只是使用输入 `person_to_be_greated` 打个招呼，如 [examples/hello_world/src/worker.go](https://github.com/conductor-oss/go-sdk/blob/main/examples/hello_world/src/worker.go) 所示。

```go
func Greet(task *model.Task) (interface{}, error) {
	return map[string]interface{}{
		"hello": "Hello, " + fmt.Sprintf("%v", task.InputData["person_to_be_greated"]),
	}, nil
}
```

要了解工作者的更多内容，请参阅 [使用 Go SDK 编写工作者](https://github.com/conductor-oss/go-sdk/blob/main/docs/workers_sdk.md)。

> [!note]
> 单个工作流可以包含用不同语言编写、部署在任意位置的任务工作者，让你的工作流实现多语言化和分布式！

### 步骤 3：运行应用 { #step-3-running-the-application }

该应用将启动 Greet 工作者（用于执行 "greet" 类型的任务），并注册在[步骤 1](#step-1-creating-the-workflow-by-code)中创建的工作流。

首先，让我们看看 [examples/hello_world/main.go](https://github.com/conductor-oss/go-sdk/blob/main/examples/hello_world/main.go) 中的变量声明。

```go

var (
	apiClient        = client.NewAPIClientFromEnv()
	taskRunner       = worker.NewTaskRunnerWithApiClient(apiClient)
	workflowExecutor = executor.NewWorkflowExecutor(apiClient)
)

```

首先我们创建一个 `APIClient` 实例。这是一个 REST 客户端。

我们需要为客户端提供正确的设置。在本示例中使用了 `client.NewAPIClientFromEnv()`，它通过读取以下环境变量来初始化一个新客户端：`CONDUCTOR_SERVER_URL`、`CONDUCTOR_AUTH_KEY` 和 `CONDUCTOR_AUTH_SECRET`。
`CONDUCTOR_CLIENT_HTTP_TIMEOUT` 用于以秒为单位配置客户端的 HTTP 超时时间。如果未设置，默认为 30 秒。

> [!tip]
> 有关高级配置选项和详细示例，请参阅 [API 客户端配置指南](https://github.com/conductor-oss/go-sdk/blob/main/docs/api_client/README.md)。

现在让我们看看 `main` 函数：

```go
func main() {
	// Start the Greet Worker. This worker will process "greet" tasks.
	taskRunner.StartWorker("greet", hello_world.Greet, 1, time.Millisecond*100)

	// This is used to register the Workflow, it's a one-time process. You can comment from here
	wf := hello_world.CreateWorkflow(workflowExecutor)
	err := wf.Register(true)
	if err != nil {
		log.Error(err.Error())
		return
	}
	// Till Here after registering the workflow

	// Start the greetings workflow 
	id, err := workflowExecutor.StartWorkflow(
		&model.StartWorkflowRequest{
			Name:    "greetings",
			Version: 1,
			Input: map[string]string{
				"name": "Gopher",
			},
		},
	)

	if err != nil {
		log.Error(err.Error())
		return
	}

	log.Info("Started workflow with Id: ", id)

	// Get a channel to monitor the workflow execution -
	// Note: This is useful in case of short duration workflows that completes in few seconds.
	channel, _ := workflowExecutor.MonitorExecution(id)
	run := <-channel
	log.Info("Output of the workflow: ", run.Output)
}
```

`taskRunner` 使用 `apiClient` 轮询任务并完成它们。它还会为我们启动工作者，并根据提供的设置处理并发和轮询间隔。

仅仅一行 `taskRunner.StartWorker("greet", hello_world.Greet, 1, time.Millisecond*100)` 就足以让我们的 Greet 工作者运行起来并处理 "greet" 类型的任务。

`workflowExecutor` 在 `apiClient` 之上提供了一层用于管理工作流的抽象。`ConductorWorkflow` 在内部使用它来注册工作流，同时也用它来启动和监控执行。

#### 使用本地 Conductor OSS 服务器运行示例：
```shell
export CONDUCTOR_SERVER_URL="http://localhost:8080/api"
cd examples
go run hello_world/main.go
```

#### 使用 [Orkes 开发者账户](https://developer.orkescloud.com) 运行示例。
```shell
export CONDUCTOR_SERVER_URL="https://developer.orkescloud.com/api"
export CONDUCTOR_AUTH_KEY="..."
export CONDUCTOR_AUTH_SECRET="..."
cd examples
go run hello_world/main.go
```

> [!note]
> Orkes Conductor 需要身份验证。[从服务器获取密钥和 secret](https://orkes.io/content/how-to-videos/access-key-and-secret) 以设置这些变量。

上述命令应产生类似如下的输出：
```shell
INFO[0000] Updated poll interval for task: greet, to: 100ms 
INFO[0000] Started 1 worker(s) for taskName greet, polling in interval of 100 ms 
INFO[0000] Started workflow with Id:14a9fcc5-3d74-11ef-83dc-acde48001122 
INFO[0000] Output of the workflow:map[Greetings:Hello, Gopher] 
```

## 已弃用的方法
SDK 客户端接口中的一些方法现已弃用。它们已被命名更加一致的新方法所取代。有关如何更新代码的详细信息，请参阅[迁移指南](https://github.com/conductor-oss/go-sdk/blob/main/docs/migration_guide.md)。
# 延伸阅读

- [使用 Go SDK 编写工作者](https://github.com/conductor-oss/go-sdk/blob/main/docs/workers_sdk.md)
- [使用 Go SDK 编写工作流](https://github.com/conductor-oss/go-sdk/blob/main/docs/workflow_sdk.md)
- [日志配置](https://github.com/conductor-oss/go-sdk/blob/main/docs/logger_sdk.md)
- [迁移指南：已弃用的方法](https://github.com/conductor-oss/go-sdk/blob/main/docs/migration_guide.md)
- [API 客户端配置](https://github.com/conductor-oss/go-sdk/blob/main/docs/api_client/README.md) - API 客户端设置、身份验证和代理配置的完整指南
- [TLS 配置指南](https://github.com/conductor-oss/go-sdk/blob/main/docs/api_client/tls_configuration.md) - 自签名证书和 mTLS 的 TLS/SSL 配置


## Examples

Browse all examples on GitHub: [conductor-oss/go-sdk/examples](https://github.com/conductor-oss/go-sdk/tree/main/examples)

| Example | Type |
|---|---|
| [Readme](https://github.com/conductor-oss/go-sdk/blob/main/examples/README.md) | file |
| [Api Gateway](https://github.com/conductor-oss/go-sdk/tree/main/examples/api_gateway) | directory |
| [Hello World](https://github.com/conductor-oss/go-sdk/tree/main/examples/hello_world) | directory |
| [Workflow](https://github.com/conductor-oss/go-sdk/tree/main/examples/workflow) | directory |
