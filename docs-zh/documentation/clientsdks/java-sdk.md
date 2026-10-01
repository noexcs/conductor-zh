---
description: "通过自动轮询、线程管理和 Spring Boot 集成，用 Java 构建 Conductor 工作者。"
source_repo: "https://github.com/conductor-oss/java-sdk"
sdk_page: java
---

# Java SDK

## 安装 SDK

SDK 需要 Java 21 及以上版本。向你的项目添加以下依赖：

**使用 Gradle：**

```gradle
dependencies {
    implementation 'org.conductoross:conductor-client:VERSION'

    // Optionally, you can also add spring module for auto configuration
    // implementation 'org.conductoross:conductor-client-spring:VERSION'
}
```

**使用 Maven：**

```xml
<dependency>
    <groupId>org.conductoross</groupId>
    <artifactId>conductor-client</artifactId>
    <version>VERSION</version>
</dependency>
```
*可选：你还可以添加 spring 模块以启用自动配置*
```xml
<dependency>
    <groupId>org.conductoross</groupId>
    <artifactId>conductor-client-spring</artifactId>
    <version>VERSION</version>
</dependency>
```


## 60 秒快速开始

**步骤 1：编写一个工作者**

工作者是实现 `Worker` 接口的 Java 类，它们会轮询 Conductor 以获取待执行的任务。

```java
public class GreetWorker implements Worker {
    
    @Override
    public String getTaskDefName() {
        return "greet";
    }

    @Override
    public TaskResult execute(Task task) {
        String name = (String) task.getInputData().get("name");
        TaskResult result = new TaskResult(task);
        result.setStatus(TaskResult.Status.COMPLETED);
        result.addOutputData("greeting", "Hello, " + name + "!");
        return result;
    }
}
```

**步骤 2：运行你的第一个工作流应用**

创建如下内容的 `Main.java`：

```java
import io.orkes.conductor.client.ApiClient;
import io.orkes.conductor.client.OrkesClients;
import com.netflix.conductor.client.automator.TaskRunnerConfigurer;
import com.netflix.conductor.common.metadata.workflow.StartWorkflowRequest;
import com.netflix.conductor.sdk.workflow.def.ConductorWorkflow;
import com.netflix.conductor.sdk.workflow.def.tasks.SimpleTask;
import com.netflix.conductor.sdk.workflow.executor.WorkflowExecutor;

import java.util.List;
import java.util.Map;

public class Main {
    public static void main(String[] args) {
        // Configure the SDK via ApiClient (enterprise-compatible path)
        ApiClient apiClient = ApiClient.builder().build();
        OrkesClients clients = new OrkesClients(apiClient);

        // Create workflow executor
        WorkflowExecutor executor = new WorkflowExecutor(apiClient, 100);

        // Build and register the workflow
        ConductorWorkflow<Map> workflow = new ConductorWorkflow<>(executor);
        workflow.setName("greetings");
        workflow.setVersion(1);

        SimpleTask greetTask = new SimpleTask("greet", "greet_ref");
        greetTask.input("name", "${workflow.input.name}");
        workflow.add(greetTask);
        workflow.registerWorkflow(true, true);

        // Start polling for tasks using OrkesTaskClient
        TaskRunnerConfigurer configurer = new TaskRunnerConfigurer.Builder(
                clients.getTaskClient(),
                List.of(new GreetWorker())
        ).withThreadCount(10).build();
        configurer.init();

        // Run the workflow using OrkesWorkflowClient
        StartWorkflowRequest request = new StartWorkflowRequest();
        request.setName("greetings");
        request.setVersion(1);
        request.setInput(Map.of("name", "Conductor"));
        String workflowId = clients.getWorkflowClient().startWorkflow(request);

        System.out.println("Started workflow: " + workflowId);
        System.out.println("View execution at: " + apiClient.getBasePath().replace("/api", "") + "/execution/" + workflowId);
    }
}
```

运行它：

```shell
./gradlew run
```

就这样——你刚刚定义了一个工作者、构建了一个工作流并执行了它。打开你所配置的 Conductor 服务器的 UI，即可检查该执行（实例）。

## 完整工作者示例

请参阅 [examples/basics/hello-world/](https://github.com/conductor-oss/java-sdk/tree/main/examples/basics/hello-world)，其中包含一个完整可运行的示例，涵盖：
- 使用 SDK 定义工作流
- 使用注解的工作者实现
- 工作流执行与监控

---

## 工作者

工作者是执行 Conductor 任务的 Java 类。实现 `Worker` 接口或使用 `@WorkerTask` 注解：

**使用 Worker 接口：**

```java
public class MyWorker implements Worker {
    
    @Override
    public String getTaskDefName() {
        return "my_task";
    }

    @Override
    public TaskResult execute(Task task) {
        // Your business logic here
        TaskResult result = new TaskResult(task);
        result.setStatus(TaskResult.Status.COMPLETED);
        result.addOutputData("result", "Task completed successfully");
        return result;
    }
}
```

**使用 @WorkerTask 注解：**

```java
public class Workers {
    
    @WorkerTask("greet")
    public String greet(@InputParam("name") String name) {
        return "Hello, " + name + "!";
    }
    
    @WorkerTask("process_data")
    public Map<String, Object> processData(@InputParam("data") Map<String, Object> data) {
        // Process and return data
        return Map.of("processed", true, "result", data);
    }
}
```

**启动工作者**：使用 `TaskRunnerConfigurer` 或 `WorkflowExecutor`：

```java
// Option 1: Using TaskRunnerConfigurer
ApiClient apiClient = ApiClient.builder().build();
OrkesClients clients = new OrkesClients(apiClient);

TaskRunnerConfigurer configurer = new TaskRunnerConfigurer.Builder(
    clients.getTaskClient(),
    List.of(new MyWorker(), new AnotherWorker())
)
.withThreadCount(10)
.build();
configurer.init();

// Option 2: Using WorkflowExecutor (auto-discovers @WorkerTask annotations)
WorkflowExecutor executor = new WorkflowExecutor(apiClient, 10);
executor.initWorkers("com.mycompany.workers");  // Package to scan for @WorkerTask
```

**工作者设计原则：**

- 工作者应是无状态的且幂等的
- 妥善处理失败场景
- 向 Conductor 汇报状态
- 快速完成执行（对长时间运行的任务使用轮询）

**工作者与 HTTP 端点：**

| 特性 | 工作者 | HTTP 端点 |
|---------|--------|---------------|
| 部署 | 内嵌于应用中 | 独立服务 |
| 可扩展性 | 水平扩展（增加更多实例） | 水平扩展（增加更多实例） |
| 延迟 | 较低（直接轮询） | 较高（网络开销） |
| 复杂度 | 简单 | 复杂（服务网格、负载均衡器） |

**了解更多：**
- [工作者 SDK 指南](https://github.com/conductor-oss/java-sdk/blob/main/docs/workers.md) — 完整的工作者框架文档
- [工作者示例](https://github.com/conductor-oss/java-sdk/blob/main/examples/) — 示例工作者实现

## 监控工作者

启用指标收集以监控工作者：

```java
// Using conductor-client-metrics module
dependencies {
    implementation 'org.conductoross:conductor-client-metrics:VERSION'
}
```

```java
// Configure metrics with Prometheus
TaskRunnerConfigurer configurer = new TaskRunnerConfigurer.Builder(taskClient, workers)
    .withThreadCount(10)
    .withMetricsCollector(new PrometheusMetricsCollector())
    .build();
```

完整指标文档请参阅 [conductor-client-metrics/README.md](https://github.com/conductor-oss/java-sdk/blob/main/conductor-client-metrics/README.md)。

## 工作流

使用 `ConductorWorkflow` 构建器在 Java 中定义工作流：

```java
ConductorWorkflow<MyInput> workflow = new ConductorWorkflow<>(executor);
workflow.setName("my_workflow");
workflow.setVersion(1);
workflow.setOwnerEmail("team@example.com");

// Add tasks
SimpleTask task1 = new SimpleTask("task1", "task1_ref");
SimpleTask task2 = new SimpleTask("task2", "task2_ref");
workflow.add(task1);
workflow.add(task2);

// Register the workflow
workflow.registerWorkflow(true, true);
```

**执行工作流：**

```java
ApiClient apiClient = ApiClient.builder().build();
OrkesClients clients = new OrkesClients(apiClient);
WorkflowClient workflowClient = clients.getWorkflowClient();

// Synchronous (start and poll for completion)
CompletableFuture<Workflow> future = workflow.execute(input);
Workflow result = future.get(30, TimeUnit.SECONDS);
System.out.println("Output: " + result.getOutput());

// Asynchronous (returns workflow ID immediately)
StartWorkflowRequest request = new StartWorkflowRequest();
request.setName("my_workflow");
request.setVersion(1);
request.setInput(Map.of("key", "value"));
String workflowId = workflowClient.startWorkflow(request);

// Dynamic execution (sends workflow definition with request)
CompletableFuture<Workflow> dynamicRun = workflow.executeDynamic(input);
```

**管理运行中的工作流：**

```java
// Get workflow status
Workflow wf = workflowClient.getWorkflow(workflowId, true);
System.out.println("Status: " + wf.getStatus());

// Pause, resume, terminate
workflowClient.pauseWorkflow(workflowId);
workflowClient.resumeWorkflow(workflowId);
workflowClient.terminateWorkflow(workflowId, "No longer needed");

// Retry and restart failed workflows
workflowClient.retryWorkflow(workflowId);
workflowClient.restartWorkflow(workflowId, false);
```

**了解更多：**
- [工作流 SDK 指南](https://github.com/conductor-oss/java-sdk/blob/main/docs/workflows.md) — 工作流即代码（Workflow-as-code）文档
- [工作流测试](https://github.com/conductor-oss/java-sdk/blob/main/docs/workflow-testing.md) — 工作流的单元测试

## 故障排查

**工作者停止轮询或崩溃：**
- 检查与 Conductor 服务器的网络连接
- 确认 `CONDUCTOR_SERVER_URL` 设置正确
- 确保线程池大小足以支撑你的工作负载
- 监控 JVM 内存和 GC 暂停

**连接被拒绝错误：**
- 确认 Conductor 服务器正在运行：`curl http://localhost:8080/health`
- 连接远程服务器时检查防火墙规则
- 对于 Orkes Conductor，确认身份验证凭据正确

**任务卡在 SCHEDULED 状态：**
- 确保工作者正在轮询正确的任务类型
- 检查 `getTaskDefName()` 与工作流中的任务名是否一致
- 确认工作者线程数足够

**工作流执行超时：**
- 在定义中增大工作流超时时间
- 检查任务是否在预期时间内完成
- 监控 Conductor 服务器日志中的错误

**Orkes Conductor 身份验证错误：**
- 确认已设置 `CONDUCTOR_AUTH_KEY` 和 `CONDUCTOR_AUTH_SECRET`
- 确保应用具有所需权限
- 检查凭据是否已过期

---

## 文件处理 { #file-handling }

二进制工作流值是不透明的 `conductor://file/<id>` 字符串。工作者注入 `org.conductoross.conductor.client.FileClient`，将句柄字符串作为任务输入接收，并将句柄字符串作为输出发布。上传和下载都是显式操作；任务运行器不会扫描工作者对象以查找文件。

每个操作都需要工作流 ID：

```java
public String upload(String workflowId, Path source);

public String upload(
        String workflowId,
        Path source,
        FileUploadOptions options);

public String upload(
        String workflowId,
        InputStream source,
        FileUploadOptions options);

public Path download(
        String workflowId,
        String fileHandleId,
        Path destination);

public FileMetadata getMetadata(
        String workflowId,
        String fileHandleId);
```

`FileUploadOptions` 支持 `fileName`、`contentType` 和可选的生产方 `taskId`。分片（Multipart）被有意排除在选项之外：`FileClient` 会根据源大小、配置的阈值和存储提供方能力自动选择。

### 上传路径（文件名自动推断）

```java
Path report = Path.of("/work/monthly-report.pdf");
String handle = fileClient.upload(workflowId, report);
```

源必须是一个可读的常规文件。其最终路径段将作为文件名。

### 带元数据上传路径

```java
String handle = fileClient.upload(
        task.getWorkflowInstanceId(),
        report,
        new FileUploadOptions()
                .setFileName("customer-report.pdf")
                .setContentType("application/pdf")
                .setTaskId(task.getTaskId()));
```

### 上传流

```java
FileUploadOptions options = new FileUploadOptions()
        .setFileName("events.ndjson")
        .setContentType("application/x-ndjson")
        .setTaskId(task.getTaskId());

try (InputStream source = eventStore.openExport()) {
    String handle = fileClient.upload(task.getWorkflowInstanceId(), source, options);
    // FileClient does not close source; this try-with-resources block owns it.
}
```

流上传需要一个安全的文件名。`FileClient` 在创建服务器记录之前，会将流缓冲到一个可重复读取的临时路径，之后删除临时文件，并且永不关闭调用方持有的流。

### 读取元数据

```java
FileMetadata metadata = fileClient.getMetadata(workflowId, handle);

System.out.printf(
        "%s: %s, %d bytes, status=%s%n",
        metadata.getFileName(),
        metadata.getContentType(),
        metadata.getFileSize(),
        metadata.getUploadStatus());
```

### 下载到路径

```java
Path destination = Path.of("/work/input.pdf");
Path downloaded = fileClient.download(workflowId, handle, destination);
```

目标路径可以是新路径，也可以是已存在的路径。客户端先下载到一个唯一的同级临时文件，仅在传输成功后才原子地替换目标。下载失败时会删除临时文件，并保持已存在的目标文件不变。

### 在工作者之间传递文件

```java
public final class RenderWorker implements Worker {
    private final FileClient fileClient;

    public RenderWorker(FileClient fileClient) {
        this.fileClient = fileClient;
    }

    @Override
    public TaskResult execute(Task task) {
        TaskResult result = new TaskResult(task);
        Path source = null;
        Path rendered = null;
        try {
            String workflowId = task.getWorkflowInstanceId();
            String sourceHandle = (String) task.getInputData().get("source");
            source = Files.createTempFile("source-", ".bin");
            rendered = Files.createTempFile("rendered-", ".pdf");
            fileClient.download(workflowId, sourceHandle, source);
            render(source, rendered);

            String renderedHandle = fileClient.upload(
                    workflowId,
                    rendered,
                    new FileUploadOptions()
                            .setContentType("application/pdf")
                            .setTaskId(task.getTaskId()));

            result.setStatus(TaskResult.Status.COMPLETED);
            result.addOutputData("rendered", renderedHandle);
        } catch (Exception e) {
            result.setStatus(TaskResult.Status.FAILED);
            result.setReasonForIncompletion(e.getMessage());
        } finally {
            deleteQuietly(source);
            deleteQuietly(rendered);
        }
        return result;
    }
}
```

工作流将 `${render.output.rendered}` 作为普通字符串映射到下一个任务。同一工作流族中的父工作流或子工作流可以读取元数据并下载该句柄。只有确切归属的工作流才能刷新或完成其上传。

### 自动分片与重试

Spring 自动配置会创建 `FileClient` 并读取以下设置：

```properties
conductor.file-client.retry-count=3
conductor.file-client.multipart-threshold=104857600
conductor.file-client.multipart-part-size=10485760
```

大于阈值的文件对 S3 和 Azure Blob 使用分片上传。GCS、本地存储和通用 HTTP(S) 签名 URL 保持单请求上传。每次重试前，客户端都会获取一个新的签名 URL；它只会对瞬态 I/O 故障、限流、签名过期和服务器错误进行重试，并在线程被中断时停止。

可运行的示例见 [Media Transcoder example](https://github.com/conductor-oss/java-sdk/tree/main/examples/file-storage/media-transcoder)，服务器配置见[文件存储](../advanced/file-storage.md)，REST 契约见[文件 API](../api/files.md)。

---

## AI 与 LLM 工作流

Conductor 支持 AI 原生工作流，包括智能体式工具调用、RAG 管道和多智能体编排。

**智能体式工作流**

构建 AI 智能体，让 LLM 动态选择并调用 Java 工作者作为工具。所有智能体式示例都位于 [`AgenticExamplesRunner.java`](https://github.com/conductor-oss/java-sdk/blob/main/examples/old/src/main/java/io/orkes/conductor/sdk/examples/agentic/AgenticExamplesRunner.java) —— 一个统一的运行器。

| Workflow | Description |
|----------|-------------|
| `llm_chat_workflow` | 使用 `LLM_CHAT_COMPLETE` 系统任务进行自动多轮问答 |
| `llm_chat_human_in_loop` | 通过 WAIT 任务暂停以等待用户输入的交互式聊天 |
| `multiagent_chat_demo` | 由主持人在两位 LLM 辩手之间路由的多智能体辩论 |
| `function_calling_workflow` | LLM 选择要调用哪个 Java 工作者，返回 JSON，由调度（dispatch）工作者执行 |
| `mcp_ai_agent` | 使用 MCP 工具的 AI 智能体（ListMcpTools → LLM 规划 → CallMcpTool → 汇总） |

**LLM 与 RAG 工作流**

| Example | Description |
|---------|-------------|
| [RagWorkflowExample.java](https://github.com/conductor-oss/java-sdk/blob/main/examples/old/src/main/java/io/orkes/conductor/sdk/examples/agentic/RagWorkflowExample.java) | 端到端 RAG：文档索引、语义搜索、答案生成 |
| [VectorDbExample.java](https://github.com/conductor-oss/java-sdk/blob/main/examples/old/src/main/java/io/orkes/conductor/sdk/examples/agentic/VectorDbExample.java) | 向量数据库操作：文本索引、嵌入生成和语义搜索 |

**在工作流中使用 LLM 任务：**

```java
// Chat completion task (LLM_CHAT_COMPLETE system task)
LlmChatComplete chatTask = new LlmChatComplete("chat_assistant", "chat_ref")
    .llmProvider("openai")
    .model("gpt-4o-mini")
    .messages(List.of(
        Map.of("role", "system", "message", "You are a helpful assistant."),
        Map.of("role", "user", "message", "${workflow.input.question}")
    ))
    .temperature(0.7)
    .maxTokens(500);

// Text completion task (LLM_TEXT_COMPLETE system task)
LlmTextComplete textTask = new LlmTextComplete("generate_text", "text_ref")
    .llmProvider("openai")
    .model("gpt-4o-mini")
    .promptName("my-prompt-template")
    .temperature(0.7);

// Document indexing for RAG (LLM_INDEX_DOCUMENT system task)
LlmIndexDocument indexTask = new LlmIndexDocument("index_doc", "index_ref")
    .vectorDb("pinecone")
    .namespace("my-docs")
    .index("knowledge-base")
    .embeddingModel("text-embedding-ada-002")
    .text("${workflow.input.document}");

// Semantic search (LLM_SEARCH_INDEX system task)
LlmSearchIndex searchTask = new LlmSearchIndex("search_docs", "search_ref")
    .vectorDb("pinecone")
    .namespace("my-docs")
    .index("knowledge-base")
    .query("${workflow.input.question}")
    .topK(5);

// MCP tool discovery (MCP_LIST_TOOLS system task — Orkes Conductor)
ListMcpTools listTools = new ListMcpTools("discover_tools", "tools_ref")
    .mcpServer("http://localhost:3001/mcp");

// MCP tool execution (MCP_CALL_TOOL system task — Orkes Conductor)
CallMcpTool callTool = new CallMcpTool("execute_tool", "tool_ref")
    .mcpServer("http://localhost:3001/mcp")
    .method("${tools_ref.output.result.method}")
    .arguments("${tools_ref.output.result.arguments}");

workflow.add(chatTask);
workflow.add(textTask);
workflow.add(indexTask);
```

运行所有智能体式示例：

```shell
export CONDUCTOR_SERVER_URL=http://localhost:8080/api
export OPENAI_API_KEY=your-key   # or ANTHROPIC_API_KEY

# Run all examples end-to-end
./gradlew :examples:run --args="--all"

# Run specific workflow
./gradlew :examples:run --args="--menu"
```

## 示例

完整目录见[示例指南](https://github.com/conductor-oss/java-sdk/blob/main/examples/README.md)。主要示例：

| Example | Description | Run |
|---------|-------------|-----|
| [Hello World](https://github.com/conductor-oss/java-sdk/tree/main/examples/basics/hello-world) | 带工作者的最简工作流 | `./gradlew :examples:run -PmainClass=com.netflix.conductor.sdk.examples.helloworld.Main` |
| [Workflow Operations](https://github.com/conductor-oss/java-sdk/tree/main/examples/old/src/main/java/io/orkes/conductor/sdk/examples/workflowops) | 暂停、恢复、终止工作流 | `./gradlew :examples:run -PmainClass=io.orkes.conductor.sdk.examples.workflowops.Main` |
| [Shipment Workflow](https://github.com/conductor-oss/java-sdk/tree/main/examples/old/src/main/java/com/netflix/conductor/sdk/examples/shipment) | 真实场景的订单处理 | `./gradlew :examples:run -PmainClass=com.netflix.conductor.sdk.examples.shipment.Main` |
| [Events](https://github.com/conductor-oss/java-sdk/tree/main/examples/old/src/main/java/com/netflix/conductor/sdk/examples/events) | 事件驱动的工作流 | `./gradlew :examples:run -PmainClass=com.netflix.conductor.sdk.examples.events.EventHandlerExample` |
| [All AI examples](https://github.com/conductor-oss/java-sdk/blob/main/examples/old/src/main/java/io/orkes/conductor/sdk/examples/agentic/AgenticExamplesRunner.java) | 所有智能体式/LLM 工作流 | `./gradlew :examples:run --args="--all"` |
| [RAG Workflow](https://github.com/conductor-oss/java-sdk/blob/main/examples/old/src/main/java/io/orkes/conductor/sdk/examples/agentic/RagWorkflowExample.java) | RAG 管道（索引 → 搜索 → 回答） | `./gradlew :examples:run -PmainClass=io.orkes.conductor.sdk.examples.agentic.RagWorkflowExample` |
| [Media Transcoder](https://github.com/conductor-oss/java-sdk/tree/main/examples/file-storage/media-transcoder) | 文件处理管道：上传视频 → 转码 → 缩略图 → 清单 | `mvn -f examples/file-storage/media-transcoder/pom.xml exec:java` |

## API 旅程示例

覆盖每个领域全部 API 的端到端示例：

| Example | APIs | Run |
|---------|------|-----|
| [Metadata Management](https://github.com/conductor-oss/java-sdk/blob/main/examples/old/src/main/java/io/orkes/conductor/sdk/examples/MetadataManagement.java) | 任务与工作流定义 | `./gradlew :examples:run -PmainClass=io.orkes.conductor.sdk.examples.MetadataManagement` |
| [Workflow Management](https://github.com/conductor-oss/java-sdk/blob/main/examples/old/src/main/java/io/orkes/conductor/sdk/examples/WorkflowManagement.java) | 启动、监控、控制工作流 | `./gradlew :examples:run -PmainClass=io.orkes.conductor.sdk.examples.WorkflowManagement` |
| [Authorization Management](https://github.com/conductor-oss/java-sdk/blob/main/examples/old/src/main/java/io/orkes/conductor/sdk/examples/AuthorizationManagement.java) | 用户、群组、权限 | `./gradlew :examples:run -PmainClass=io.orkes.conductor.sdk.examples.AuthorizationManagement` |
| [Scheduler Management](https://github.com/conductor-oss/java-sdk/blob/main/examples/old/src/main/java/io/orkes/conductor/sdk/examples/SchedulerManagement.java) | 工作流调度 | `./gradlew :examples:run -PmainClass=io.orkes.conductor.sdk.examples.SchedulerManagement` |

## 文档

| Document | Description |
|----------|-------------|
| [工作者 SDK](https://github.com/conductor-oss/java-sdk/blob/main/docs/workers.md) | 完整的工作者框架指南 |
| [工作流 SDK](https://github.com/conductor-oss/java-sdk/blob/main/docs/workflows.md) | 工作流即代码文档 |
| [测试框架](https://github.com/conductor-oss/java-sdk/blob/main/docs/workflow-testing.md) | 工作流和工作者的单元测试 |
| [Conductor 客户端](https://github.com/conductor-oss/java-sdk/blob/main/conductor-client/README.md) | HTTP 客户端库文档 |
| [客户端指标](https://github.com/conductor-oss/java-sdk/blob/main/conductor-client-metrics/README.md) | Prometheus 指标收集 |
| [Spring 集成](https://github.com/conductor-oss/java-sdk/blob/main/conductor-client-spring/README.md) | Spring Boot 自动配置 |
| [示例](https://github.com/conductor-oss/java-sdk/blob/main/examples/README.md) | 完整示例目录 |

## 支持

- 针对 SDK 的 bug、问题和功能请求，请[提交 Issue（SDK）](https://github.com/conductor-sdk/conductor-java-sdk/issues)
- 针对 Conductor OSS 服务器的问题，请[提交 Issue（Conductor server）](https://github.com/conductor-oss/conductor/issues)
- 参与社区讨论和寻求帮助，请[加入 Conductor Slack](https://join.slack.com/t/orkes-conductor/shared_invite/zt-2vdbx239s-Eacdyqya9giNLHfrCavfaA)
- 问答交流请见 [Orkes 社区论坛](https://community.orkes.io/)

## 许可证

Apache 2.0


## Examples

Browse all examples on GitHub: [conductor-oss/java-sdk/examples](https://github.com/conductor-oss/java-sdk/tree/main/examples)

| Example | Type |
|---|---|
| [Readme](https://github.com/conductor-oss/java-sdk/blob/main/examples/README.md) | file |
| [Examples](https://github.com/conductor-oss/java-sdk/tree/main/examples) | directory |
