---
description: "使用类型安全的任务定义和异步工作流管理，用 Rust 构建 Conductor 工作者。"
source_repo: "https://github.com/conductor-oss/rust-sdk"
sdk_page: rust
---

# Rust SDK

## 安装 SDK

向你的 `Cargo.toml` 添加以下内容：

```toml
[dependencies]
conductor = "VERSION"
tokio = { version = "1", features = ["full"] }
```

如需 `#[worker]` 宏（类似于 Python 的 `@worker_task` 装饰器）：

```toml
[dependencies]
conductor = { version = "VERSION", features = ["macros"] }
conductor-macros = "VERSION"
tokio = { version = "1", features = ["full"] }
```

## 60 秒快速开始

**步骤 1：创建工作流**

工作流是引用任务类型（例如名为 `greet` 的 SIMPLE 任务）的定义。我们将构建一个名为
`greetings` 的工作流，它运行一个任务并返回其输出。

```rust
use conductor::models::{WorkflowDef, WorkflowTask};

fn greetings_workflow() -> WorkflowDef {
    WorkflowDef::new("greetings")
        .with_version(1)
        .with_task(
            WorkflowTask::simple("greet", "greet_ref")
                .with_input_param("name", "${workflow.input.name}")
        )
        .with_output_param("result", "${greet_ref.output.result}")
}
```

**步骤 2：编写工作者**

工作者是使用 `#[worker]` 装饰的 Rust 函数，它们轮询 Conductor 以获取任务并执行。

```rust
use conductor_macros::worker;

#[worker(name = "greet")]
async fn greet(name: String) -> String {
    format!("Hello {}", name)
}
```

**步骤 3：运行你的第一个工作流应用**

创建如下内容的 `main.rs`：

```rust
use conductor::{
    client::ConductorClient,
    configuration::Configuration,
    models::{StartWorkflowRequest, WorkflowDef, WorkflowTask},
    worker::TaskHandler,
};
use conductor_macros::worker;

// A worker is any Rust function with the #[worker] macro.
#[worker(name = "greet")]
async fn greet(name: String) -> String {
    format!("Hello {}", name)
}

fn greetings_workflow() -> WorkflowDef {
    WorkflowDef::new("greetings")
        .with_version(1)
        .with_task(
            WorkflowTask::simple("greet", "greet_ref")
                .with_input_param("name", "${workflow.input.name}")
        )
        .with_output_param("result", "${greet_ref.output.result}")
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Configure the SDK (reads CONDUCTOR_SERVER_URL / CONDUCTOR_AUTH_* from env).
    let config = Configuration::default();
    let client = ConductorClient::new(config.clone())?;

    // Register the workflow
    let workflow = greetings_workflow();
    client.metadata_client()
        .register_or_update_workflow_def(&workflow, true)
        .await?;

    // Start polling for tasks
    let mut task_handler = TaskHandler::new(config.clone())?;
    task_handler.add_worker(greet_worker());
    task_handler.start().await?;

    // Run the workflow and get the result
    let run = client.workflow_client()
        .execute_workflow(
            &StartWorkflowRequest::new("greetings")
                .with_version(1)
                .with_input_value("name", "Conductor"),
            std::time::Duration::from_secs(10),
        )
        .await?;

    println!("result: {:?}", run.output.get("result"));
    println!("execution: {}/execution/{}", config.ui_host, run.workflow_id);

    task_handler.stop().await?;
    Ok(())
}
```

运行它：

```shell
cargo run
```

详情见 [rust-sdk README](https://github.com/conductor-oss/rust-sdk)。

就这样——你刚刚定义了一个工作者、构建了一个工作流并执行了它。打开你所配置的 Conductor 服务器的 UI，即可检查该执行（实例）。

## 完整工作者示例

示例包含同步 + 异步工作者、指标和长时间运行任务。

见 [examples/worker_example.rs](https://github.com/conductor-oss/rust-sdk/blob/main/examples/worker_example.rs)

---

## 工作者

工作者是执行 Conductor 任务的 Rust 函数。使用 `#[worker]` 宏或 `FnWorker` 来：

- 将其注册为工作者（由 `TaskHandler` 自动发现）
- 将其用作工作流任务（以 `task_ref_name=...` 的方式调用）

注意：工作者也可被 LLM 用于工具调用（见 [AI 与 LLM 工作流](#ai-llm-workflows)）。

```rust
use conductor_macros::worker;

#[worker(name = "greet")]
async fn greet(name: String) -> String {
    format!("Hello {}", name)
}
```

**使用 FnWorker（基于闭包）：**

```rust
use conductor::worker::{FnWorker, WorkerOutput};

let greetings_worker = FnWorker::new("greetings", |task| async move {
    let name = task.get_input_string("name").unwrap_or_default();
    Ok(WorkerOutput::completed_with_result(format!("Hello, {}", name)))
})
.with_thread_count(10)
.with_poll_interval_millis(100);
```

**启动工作者**：使用 `TaskHandler`：

```rust
use conductor::{
    configuration::Configuration,
    worker::TaskHandler,
};

let config = Configuration::default();
let mut task_handler = TaskHandler::new(config)?;
task_handler.add_worker(greet_worker());

task_handler.start().await?;

// Wait for shutdown signal
tokio::signal::ctrl_c().await?;

task_handler.stop().await?;
```

**工作者配置**

工作者支持分层环境变量配置——全局设置可按工作者覆盖：

```shell
# Global (all workers)
export CONDUCTOR_WORKER_ALL_POLL_INTERVAL_MILLIS=250
export CONDUCTOR_WORKER_ALL_THREAD_COUNT=20
export CONDUCTOR_WORKER_ALL_DOMAIN=production

# Per-worker override
export CONDUCTOR_WORKER_GREETINGS_THREAD_COUNT=50
```

所有选项见 [WORKER_CONFIGURATION.md](https://github.com/conductor-oss/rust-sdk/blob/main/WORKER_CONFIGURATION.md)。

## 监控工作者

启用 Prometheus 指标：

```rust
use conductor::metrics::MetricsSettings;
use conductor::worker::TaskHandler;

let mut task_handler = TaskHandler::new(config)?;
task_handler.enable_metrics(
    MetricsSettings::new()
        .with_http_port(9090)
);

task_handler.start().await?;
// Metrics at http://localhost:9090/metrics
```

详情见 [rust-sdk README](https://github.com/conductor-oss/rust-sdk)。

**了解更多：**
- [工作者指南](https://github.com/conductor-oss/rust-sdk/blob/main/docs/WORKER.md) — 所有工作者模式（函数、闭包、宏、异步）
- [工作者配置](https://github.com/conductor-oss/rust-sdk/blob/main/WORKER_CONFIGURATION.md) — 环境变量配置系统

## 工作流

使用构建器模式在 Rust 中定义工作流以串联任务：

```rust
use conductor::{
    client::ConductorClient,
    configuration::Configuration,
    models::{WorkflowDef, WorkflowTask},
};

let config = Configuration::default();
let client = ConductorClient::new(config)?;
let metadata_client = client.metadata_client();

let workflow = WorkflowDef::new("greetings")
    .with_version(1)
    .with_task(
        WorkflowTask::simple("greet", "greet_ref")
            .with_input_param("name", "${workflow.input.name}")
    )
    .with_output_param("result", "${greet_ref.output.result}");

// Registering is required if you want to start/execute by name+version
metadata_client.register_or_update_workflow_def(&workflow, true).await?;
```

**执行工作流：**

```rust
use conductor::models::StartWorkflowRequest;
use std::time::Duration;

// Asynchronous (returns workflow ID immediately)
let request = StartWorkflowRequest::new("greetings")
    .with_version(1)
    .with_input_value("name", "Orkes");
let workflow_id = workflow_client.start_workflow(&request).await?;

// Synchronous (waits for completion)
let run = workflow_client
    .execute_workflow(&request, Duration::from_secs(10))
    .await?;
println!("{:?}", run.output);
```

**管理运行中的工作流并发送信号：**

```rust
workflow_client.pause_workflow(&workflow_id).await?;
workflow_client.resume_workflow(&workflow_id).await?;
workflow_client.terminate_workflow(&workflow_id, Some("no longer needed"), false).await?;
workflow_client.retry_workflow(&workflow_id, false).await?;
workflow_client.restart_workflow(&workflow_id, false).await?;
```

**了解更多：**
- [工作流管理](https://github.com/conductor-oss/rust-sdk/blob/main/docs/WORKFLOW.md) — 启动、暂停、恢复、终止、重试、搜索
- [元数据管理](https://github.com/conductor-oss/rust-sdk/blob/main/docs/METADATA.md) — 任务与工作流定义

## 故障排查

- **工作者停止轮询**：`TaskHandler` 会监控工作者。使用 `task_handler.is_healthy()` 进行健康检查。
- **连接问题**：确认 `CONDUCTOR_SERVER_URL` 正确且服务器正在运行。
- **身份验证失败**：对于 Orkes Conductor，确保 `CONDUCTOR_AUTH_KEY` 和 `CONDUCTOR_AUTH_SECRET` 有效。

---

## AI 与 LLM 工作流 { #ai-llm-workflows }

Conductor 支持 AI 原生工作流，包括智能体式工具调用、RAG 管道和多智能体编排。

**智能体式工作流**

构建 AI 智能体，让 LLM 动态选择并调用 Rust 工作者作为工具。所有示例见 [examples/](https://github.com/conductor-oss/rust-sdk/blob/main/examples/)。

| Example | Description |
|---------|-------------|
| [llm_chat_example.rs](https://github.com/conductor-oss/rust-sdk/blob/main/examples/llm_chat_example.rs) | 两个 LLM 之间的自动化多轮科学问答 |
| [llm_chat_human_in_loop.rs](https://github.com/conductor-oss/rust-sdk/blob/main/examples/llm_chat_human_in_loop.rs) | 通过 WAIT 任务暂停以等待用户输入的交互式聊天 |
| [multiagent_chat.rs](https://github.com/conductor-oss/rust-sdk/blob/main/examples/multiagent_chat.rs) | 专家、批评者和综合者之间的多智能体讨论 |
| [function_calling_example.rs](https://github.com/conductor-oss/rust-sdk/blob/main/examples/function_calling_example.rs) | LLM 根据用户查询选择调用哪个函数 |
| [agentic_workflow.rs](https://github.com/conductor-oss/rust-sdk/blob/main/examples/agentic_workflow.rs) | 带工具调用和基于 switch 路由的 AI 智能体 |

**LLM 与 RAG 工作流**

| Example | Description |
|---------|-------------|
| [rag_workflow.rs](https://github.com/conductor-oss/rust-sdk/blob/main/examples/rag_workflow.rs) | 端到端 RAG：文本索引、语义搜索、答案生成 |
| [vector_db_example.rs](https://github.com/conductor-oss/rust-sdk/blob/main/examples/vector_db_example.rs) | 带嵌入生成的向量数据库操作 |

```shell
# Automated multi-turn chat
cargo run --example llm_chat_example

# Multi-agent discussion
cargo run --example multiagent_chat

# RAG pipeline
cargo run --example rag_workflow
```

## 示例

完整目录见 examples 目录。主要示例：

| Example | Description | Run |
|---------|-------------|-----|
| [worker_example.rs](https://github.com/conductor-oss/rust-sdk/blob/main/examples/worker_example.rs) | 端到端：同步 + 异步工作者，指标 | `cargo run --example worker_example` |
| [hello_world.rs](https://github.com/conductor-oss/rust-sdk/blob/main/examples/hello_world.rs) | 最简 hello world | `cargo run --example hello_world` |
| [dynamic_workflow.rs](https://github.com/conductor-oss/rust-sdk/blob/main/examples/dynamic_workflow.rs) | 以编程方式构建工作流 | `cargo run --example dynamic_workflow` |
| [llm_chat_example.rs](https://github.com/conductor-oss/rust-sdk/blob/main/examples/llm_chat_example.rs) | AI 多轮聊天 | `cargo run --example llm_chat_example` |
| [rag_workflow.rs](https://github.com/conductor-oss/rust-sdk/blob/main/examples/rag_workflow.rs) | RAG 管道 | `cargo run --example rag_workflow` |
| [task_context_example.rs](https://github.com/conductor-oss/rust-sdk/blob/main/examples/task_context_example.rs) | 使用 TaskContext 的长时间运行任务 | `cargo run --example task_context_example` |
| [workflow_ops.rs](https://github.com/conductor-oss/rust-sdk/blob/main/examples/workflow_ops.rs) | 暂停、恢复、终止工作流 | `cargo run --example workflow_ops` |
| [test_workflows.rs](https://github.com/conductor-oss/rust-sdk/blob/main/examples/test_workflows.rs) | 工作流单元测试 | `cargo run --example test_workflows` |
| [kitchensink.rs](https://github.com/conductor-oss/rust-sdk/blob/main/examples/kitchensink.rs) | 所有任务类型（HTTP、JS、JQ、Switch） | `cargo run --example kitchensink` |

## API 旅程示例

覆盖每个领域全部 API 的端到端示例：

| Example | APIs | Run |
|---------|------|-----|
| [authorization_example.rs](https://github.com/conductor-oss/rust-sdk/blob/main/examples/authorization_example.rs) | 授权 API | `cargo run --example authorization_example` |
| [metadata_journey.rs](https://github.com/conductor-oss/rust-sdk/blob/main/examples/metadata_journey.rs) | 元数据 API | `cargo run --example metadata_journey` |
| [schedule_journey.rs](https://github.com/conductor-oss/rust-sdk/blob/main/examples/schedule_journey.rs) | 调度 API | `cargo run --example schedule_journey` |
| [prompt_journey.rs](https://github.com/conductor-oss/rust-sdk/blob/main/examples/prompt_journey.rs) | 提示（Prompt）API | `cargo run --example prompt_journey` |

## 文档

| Document | Description |
|----------|-------------|
| [工作者指南](https://github.com/conductor-oss/rust-sdk/blob/main/docs/WORKER.md) | 所有工作者模式（函数、闭包、宏、异步） |
| [工作者配置](https://github.com/conductor-oss/rust-sdk/blob/main/WORKER_CONFIGURATION.md) | 分层环境变量配置 |
| [工作流管理](https://github.com/conductor-oss/rust-sdk/blob/main/docs/WORKFLOW.md) | 启动、暂停、恢复、终止、重试、搜索 |
| [任务管理](https://github.com/conductor-oss/rust-sdk/blob/main/docs/TASK_MANAGEMENT.md) | 任务操作 |
| [元数据](https://github.com/conductor-oss/rust-sdk/blob/main/docs/METADATA.md) | 任务与工作流定义 |
| [授权](https://github.com/conductor-oss/rust-sdk/blob/main/docs/AUTHORIZATION.md) | 用户、群组、应用、权限 |
| [调度](https://github.com/conductor-oss/rust-sdk/blob/main/docs/SCHEDULE.md) | 工作流调度 |
| [密钥](https://github.com/conductor-oss/rust-sdk/blob/main/docs/SECRET_MANAGEMENT.md) | 密钥存储 |
| [提示词](https://github.com/conductor-oss/rust-sdk/blob/main/docs/PROMPT.md) | AI/LLM 提示词模板 |
| [集成](https://github.com/conductor-oss/rust-sdk/blob/main/docs/INTEGRATION.md) | AI/LLM 提供商集成 |
| [指标](https://github.com/conductor-oss/rust-sdk) | Prometheus 指标收集 |

## 支持

- 针对 SDK 的 bug、问题和功能请求，请[提交 Issue（SDK）](https://github.com/conductor-oss/rust-sdk/issues)
- 针对 Conductor OSS 服务器的问题，请[提交 Issue（Conductor server）](https://github.com/conductor-oss/conductor/issues)
- 参与社区讨论和寻求帮助，请[加入 Conductor Slack](https://join.slack.com/t/orkes-conductor/shared_invite/zt-2vdbx239s-Eacdyqya9giNLHfrCavfaA)
- 问答交流请见 [Orkes 社区论坛](https://community.orkes.io/)

## 许可证

Apache 2.0


## Examples

Browse all examples on GitHub: [conductor-oss/rust-sdk/examples](https://github.com/conductor-oss/rust-sdk/tree/main/examples)

| Example | Type |
|---|---|
| [Agentic Workflow](https://github.com/conductor-oss/rust-sdk/blob/main/examples/agentic_workflow.rs) | file |
| [Async Workers](https://github.com/conductor-oss/rust-sdk/blob/main/examples/async_workers.rs) | file |
| [Authorization Example](https://github.com/conductor-oss/rust-sdk/blob/main/examples/authorization_example.rs) | file |
| [Connection Config Example](https://github.com/conductor-oss/rust-sdk/blob/main/examples/connection_config_example.rs) | file |
| [Dynamic Workflow](https://github.com/conductor-oss/rust-sdk/blob/main/examples/dynamic_workflow.rs) | file |
| [Event Listener Example](https://github.com/conductor-oss/rust-sdk/blob/main/examples/event_listener_example.rs) | file |
| [Fork Join Script Example](https://github.com/conductor-oss/rust-sdk/blob/main/examples/fork_join_script_example.rs) | file |
| [Function Calling Example](https://github.com/conductor-oss/rust-sdk/blob/main/examples/function_calling_example.rs) | file |
| [Hello World](https://github.com/conductor-oss/rust-sdk/blob/main/examples/hello_world.rs) | file |
| [Http Poll Example](https://github.com/conductor-oss/rust-sdk/blob/main/examples/http_poll_example.rs) | file |
| [Kitchensink](https://github.com/conductor-oss/rust-sdk/blob/main/examples/kitchensink.rs) | file |
| [Kitchensink Workers](https://github.com/conductor-oss/rust-sdk/blob/main/examples/kitchensink_workers.rs) | file |
| [Llm Chat Example](https://github.com/conductor-oss/rust-sdk/blob/main/examples/llm_chat_example.rs) | file |
| [Llm Chat Human In Loop](https://github.com/conductor-oss/rust-sdk/blob/main/examples/llm_chat_human_in_loop.rs) | file |
| [Metadata Journey](https://github.com/conductor-oss/rust-sdk/blob/main/examples/metadata_journey.rs) | file |
| [Metrics Example](https://github.com/conductor-oss/rust-sdk/blob/main/examples/metrics_example.rs) | file |
| [Multiagent Chat](https://github.com/conductor-oss/rust-sdk/blob/main/examples/multiagent_chat.rs) | file |
| [Openai Helloworld](https://github.com/conductor-oss/rust-sdk/blob/main/examples/openai_helloworld.rs) | file |
| [Prompt Journey](https://github.com/conductor-oss/rust-sdk/blob/main/examples/prompt_journey.rs) | file |
| [Rag Workflow](https://github.com/conductor-oss/rust-sdk/blob/main/examples/rag_workflow.rs) | file |
| [Schedule Journey](https://github.com/conductor-oss/rust-sdk/blob/main/examples/schedule_journey.rs) | file |
| [Secret Example](https://github.com/conductor-oss/rust-sdk/blob/main/examples/secret_example.rs) | file |
| [Sync State Update Example](https://github.com/conductor-oss/rust-sdk/blob/main/examples/sync_state_update_example.rs) | file |
| [Task Configure](https://github.com/conductor-oss/rust-sdk/blob/main/examples/task_configure.rs) | file |
| [Task Context Example](https://github.com/conductor-oss/rust-sdk/blob/main/examples/task_context_example.rs) | file |
| [Task Status Audit Example](https://github.com/conductor-oss/rust-sdk/blob/main/examples/task_status_audit_example.rs) | file |
| [Task Workers](https://github.com/conductor-oss/rust-sdk/blob/main/examples/task_workers.rs) | file |
| [Test Workflows](https://github.com/conductor-oss/rust-sdk/blob/main/examples/test_workflows.rs) | file |
| [Vector Db Example](https://github.com/conductor-oss/rust-sdk/blob/main/examples/vector_db_example.rs) | file |
| [Wait For Webhook Example](https://github.com/conductor-oss/rust-sdk/blob/main/examples/wait_for_webhook_example.rs) | file |
| [Worker Config Example](https://github.com/conductor-oss/rust-sdk/blob/main/examples/worker_config_example.rs) | file |
| [Worker Example](https://github.com/conductor-oss/rust-sdk/blob/main/examples/worker_example.rs) | file |
| [Worker Macro Example](https://github.com/conductor-oss/rust-sdk/blob/main/examples/worker_macro_example.rs) | file |
| [Workflow Ops](https://github.com/conductor-oss/rust-sdk/blob/main/examples/workflow_ops.rs) | file |
| [Workflow Rerun Example](https://github.com/conductor-oss/rust-sdk/blob/main/examples/workflow_rerun_example.rs) | file |
| [Workflow Status Listener](https://github.com/conductor-oss/rust-sdk/blob/main/examples/workflow_status_listener.rs) | file |
