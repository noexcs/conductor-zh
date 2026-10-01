---
description: "使用官方生成的 SDK 以 C#/.NET 构建 Conductor 工作者和客户端。"
source_repo: "https://github.com/conductor-oss/csharp-sdk"
sdk_page: csharp
---

# C# SDK

## 安装 SDK

```shell
dotnet add package conductor-csharp --version VERSION
```

## 配置工作流客户端

SDK 快速开始示例从环境变量中配置端点，并使用工作流执行器 API：

```csharp
using Conductor.Client;
using Conductor.Definition;
using Conductor.Definition.TaskType;
using Conductor.Executor;

var configuration = new Configuration {
    BasePath = Environment.GetEnvironmentVariable("CONDUCTOR_SERVER_URL")
        ?? "http://localhost:8080/api"
};

var workflow = new ConductorWorkflow()
    .WithName("greetings")
    .WithVersion(1);

var greetTask = new SimpleTask("greet", "greet_ref")
    .WithInput("name", workflow.Input("name"));
workflow.WithTask(greetTask);

var executor = new WorkflowExecutor(configuration);
executor.RegisterWorkflow(workflow, overwrite: true);
var workflowId = executor.StartWorkflow(new StartWorkflowRequest {
    Name = "greetings",
    Version = 1,
    Input = new Dictionary<string, object> { ["name"] = "Conductor" }
});
```

对于 Orkes 身份验证，SDK 提供了 `Configuration.AuthenticationSettings`；在构建客户端之前，请基于 `CONDUCTOR_AUTH_KEY` 和 `CONDUCTOR_AUTH_SECRET` 创建一个 `OrkesAuthenticationSettings`。它不会自动进行该环境变量映射。有关其身份验证和工作者示例，请参阅[上游 SDK README](https://github.com/conductor-oss/csharp-sdk#configurations)。
