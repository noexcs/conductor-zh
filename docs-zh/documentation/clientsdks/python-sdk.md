---
description: "使用基于装饰器的任务定义、异步支持和工作流管理，用 Python 构建 Conductor 工作者。"
source_repo: "https://github.com/conductor-oss/python-sdk"
sdk_page: python
---

# Python SDK

## 安装 SDK

```shell
pip install conductor-python
```

## 60 秒快速开始

**步骤 1：创建工作流**

工作流是引用任务类型（例如名为 `greet` 的 SIMPLE 任务）的定义。我们将构建一个名为
`greetings` 的工作流，它运行一个任务并返回其输出。

假设你已有一个 `WorkflowExecutor`（`executor`）和一个工作者任务（`greet`）：

```python
from conductor.client.workflow.conductor_workflow import ConductorWorkflow

workflow = ConductorWorkflow(name='greetings', version=1, executor=executor)
greet_task = greet(task_ref_name='greet_ref', name=workflow.input('name'))
workflow >> greet_task
workflow.output_parameters({'result': greet_task.output('result')})
workflow.register(overwrite=True)
```

**步骤 2：编写一个工作者**

工作者就是使用 `@worker_task` 装饰的 Python 函数，它们轮询 Conductor 以获取任务并执行。

```python
from conductor.client.worker.worker_task import worker_task

# register_task_def=True is convenient for local dev quickstarts; in production, manage task definitions separately.
@worker_task(task_definition_name='greet', register_task_def=True)
def greet(name: str) -> str:
    return f'Hello {name}'
```

**步骤 3：运行你的第一个工作流应用**

创建如下内容的 `quickstart.py`：

```python
from conductor.client.automator.task_handler import TaskHandler
from conductor.client.configuration.configuration import Configuration
from conductor.client.orkes_clients import OrkesClients
from conductor.client.workflow.conductor_workflow import ConductorWorkflow
from conductor.client.worker.worker_task import worker_task


# A worker is any Python function.
@worker_task(task_definition_name='greet', register_task_def=True)
def greet(name: str) -> str:
    return f'Hello {name}'


def main():
    # Configure the SDK (reads CONDUCTOR_SERVER_URL / CONDUCTOR_AUTH_* from env).
    config = Configuration()

    clients = OrkesClients(configuration=config)
    executor = clients.get_workflow_executor()

    # Build a workflow with the >> operator.
    workflow = ConductorWorkflow(name='greetings', version=1, executor=executor)
    greet_task = greet(task_ref_name='greet_ref', name=workflow.input('name'))
    workflow >> greet_task
    workflow.output_parameters({'result': greet_task.output('result')})
    workflow.register(overwrite=True)

    # Start polling for tasks (one worker subprocess per worker function).
    with TaskHandler(configuration=config, scan_for_annotated_workers=True) as task_handler:
        task_handler.start_processes()

        # Run the workflow and get the result.
        run = executor.execute(name='greetings', version=1, workflow_input={'name': 'Conductor'})
        print(f'result: {run.output["result"]}')
        print(f'execution: {config.ui_host}/execution/{run.workflow_id}')


if __name__ == '__main__':
    main()
```

运行它：

```shell
python quickstart.py
```

有关可选的 HTTP/2 配置，请参阅[工作者配置](https://github.com/conductor-oss/python-sdk/blob/main/WORKER_CONFIGURATION.md)指南。

就这样——你刚刚定义了一个工作者、构建了一个工作流并执行了它。打开你所配置的 Conductor 服务器的 UI，即可检查该执行（实例）。

---

## 功能展示

### 工作者：同步与异步

SDK 会根据你的函数签名自动选择正确的运行器——同步函数使用 `TaskRunner`（线程池），异步函数使用 `AsyncTaskRunner`（事件循环）。

```python
from conductor.client.worker.worker_task import worker_task

# Sync worker — for CPU-bound work (uses ThreadPoolExecutor)
@worker_task(task_definition_name='process_image', thread_count=4)
def process_image(image_url: str) -> dict:
    import PIL.Image, io, requests
    img = PIL.Image.open(io.BytesIO(requests.get(image_url).content))
    img.thumbnail((256, 256))
    return {'width': img.width, 'height': img.height}


# Async worker — for I/O-bound work (uses AsyncTaskRunner, no thread overhead)
@worker_task(task_definition_name='fetch_data', thread_count=50)
async def fetch_data(url: str) -> dict:
    import httpx
    async with httpx.AsyncClient() as client:
        resp = await client.get(url)
    return resp.json()
```

使用 `TaskHandler` 启动工作者——它会自动发现 `@worker_task` 函数，并为每个工作者派生一个子进程：

```python
from conductor.client.automator.task_handler import TaskHandler
from conductor.client.configuration.configuration import Configuration

config = Configuration()
with TaskHandler(configuration=config, scan_for_annotated_workers=True) as task_handler:
    task_handler.start_processes()
    task_handler.join_processes()  # blocks forever (workers poll continuously)
```

完整示例见 [examples/worker_example.py](https://github.com/conductor-oss/python-sdk/blob/main/examples/worker_example.py) 和 [examples/workers_e2e.py](https://github.com/conductor-oss/python-sdk/blob/main/examples/workers_e2e.py)。

### 包含 HTTP 调用和等待的工作流

将自定义工作者与内置系统任务串联——HTTP 调用、等待、JavaScript、JQ 转换——全部在一个工作流中：

```python
from conductor.client.workflow.conductor_workflow import ConductorWorkflow
from conductor.client.workflow.task.http_task import HttpTask
from conductor.client.workflow.task.wait_task import WaitTask

workflow = ConductorWorkflow(name='order_pipeline', version=1, executor=executor)

# Custom worker task
validate = validate_order(task_ref_name='validate', order_id=workflow.input('order_id'))

# Built-in HTTP task — call any API, no worker needed
charge_payment = HttpTask(task_ref_name='charge_payment', http_input={
    'uri': 'https://api.stripe.com/v1/charges',
    'method': 'POST',
    'headers': {'Authorization': ['Bearer ${workflow.input.stripe_key}']},
    'body': {'amount': '${validate.output.amount}'}
})

# Built-in Wait task — pause the workflow for 10 seconds
cool_down = WaitTask(task_ref_name='cool_down', wait_for_seconds=10)

# Another custom worker task
notify = send_notification(task_ref_name='notify', message='Order complete')

# Chain with >> operator
workflow >> validate >> charge_payment >> cool_down >> notify

# Execute synchronously and wait for the result
result = workflow.execute(workflow_input={'order_id': 'ORD-123', 'stripe_key': 'sk_test_...'})
print(result.output)
```

所有任务类型（HTTP、JavaScript、JQ、Switch、Terminate）见 [examples/kitchensink.py](https://github.com/conductor-oss/python-sdk/blob/main/examples/kitchensink.py)，生命周期操作见 [examples/workflow_ops.py](https://github.com/conductor-oss/python-sdk/blob/main/examples/workflow_ops.py)。

### 使用 TaskContext 的长时间运行任务

对于耗时数分钟甚至数小时的任务（批处理、ML 训练、外部审批），使用 `TaskContext` 汇报进度并增量轮询：

```python
from typing import Union
from conductor.client.worker.worker_task import worker_task
from conductor.client.context.task_context import get_task_context, TaskInProgress

@worker_task(task_definition_name='batch_job')
def batch_job(batch_id: str) -> Union[dict, TaskInProgress]:
    ctx = get_task_context()
    ctx.add_log(f"Processing batch {batch_id}, poll #{ctx.get_poll_count()}")

    if ctx.get_poll_count() < 3:
        # Not done yet — re-queue and check again in 30 seconds
        return TaskInProgress(callback_after_seconds=30, output={'progress': ctx.get_poll_count() * 33})

    # Done after 3 polls
    return {'status': 'completed', 'batch_id': batch_id}
```

`TaskContext` 还提供了访问任务元数据、重试次数、工作流 ID 的能力，以及添加可在 Conductor UI 中查看的日志的能力。

所有模式（轮询、感知重试的逻辑、异步上下文、输入访问）见 [examples/task_context_example.py](https://github.com/conductor-oss/python-sdk/blob/main/examples/task_context_example.py)。

### 使用指标进行监控

只需一项设置即可启用 Prometheus 指标——SDK 暴露轮询次数、执行时间、错误率和 HTTP 延迟：

```python
from conductor.client.automator.task_handler import TaskHandler
from conductor.client.configuration.configuration import Configuration
from conductor.client.configuration.settings.metrics_settings import MetricsSettings

config = Configuration()
metrics = MetricsSettings(directory='/tmp/conductor-metrics', http_port=8000)

with TaskHandler(configuration=config, metrics_settings=metrics, scan_for_annotated_workers=True) as task_handler:
    task_handler.start_processes()
    task_handler.join_processes()
```

```shell
# Prometheus-compatible endpoint
curl http://localhost:8000/metrics
```

所有被跟踪指标的详情见 [examples/metrics_example.py](https://github.com/conductor-oss/python-sdk/blob/main/examples/metrics_example.py) 和 [METRICS.md](https://github.com/conductor-oss/python-sdk/blob/main/METRICS.md)。

### 管理工作流执行

完整的生命周期控制——启动、执行、暂停、恢复、终止、重试、重启、重新运行、发送信号和搜索：

```python
from conductor.client.configuration.configuration import Configuration
from conductor.client.http.models import StartWorkflowRequest, RerunWorkflowRequest, TaskResult
from conductor.client.orkes_clients import OrkesClients

config = Configuration()
clients = OrkesClients(configuration=config)
workflow_client = clients.get_workflow_client()
task_client = clients.get_task_client()
executor = clients.get_workflow_executor()

# Start async (returns workflow ID immediately)
workflow_id = executor.start_workflow(StartWorkflowRequest(name='my_workflow', input={'key': 'value'}))

# Execute sync (blocks until workflow completes)
result = executor.execute(name='my_workflow', version=1, workflow_input={'key': 'value'})

# Lifecycle management
workflow_client.pause_workflow(workflow_id)
workflow_client.resume_workflow(workflow_id)
workflow_client.terminate_workflow(workflow_id, reason='no longer needed')
workflow_client.retry_workflow(workflow_id)          # retry from last failed task
workflow_client.restart_workflow(workflow_id)         # restart from the beginning
workflow_client.rerun_workflow(workflow_id,           # rerun from a specific task
    RerunWorkflowRequest(re_run_from_task_id=task_id))

# Send a signal to a waiting workflow (complete a WAIT task externally)
task_client.update_task(TaskResult(
    workflow_instance_id=workflow_id,
    task_id=wait_task_id,
    status='COMPLETED',
    output_data={'approved': True}
))

# Search workflows
results = workflow_client.search(query='status IN (RUNNING) AND correlationId = "order-123"')
```

每个操作的完整演练见 [examples/workflow_ops.py](https://github.com/conductor-oss/python-sdk/blob/main/examples/workflow_ops.py)。

---

## AI 与 LLM 工作流

Conductor 支持 AI 原生工作流，包括智能体式工具调用、RAG 管道和多智能体编排。

**智能体式工作流**

构建 AI 智能体，让 LLM 动态选择并调用 Python 工作者作为工具。所有示例见 [examples/agentic_workflows/](https://github.com/conductor-oss/python-sdk/blob/main/examples/agentic_workflows/)。

| Example | Description |
|---------|-------------|
| [llm_chat.py](https://github.com/conductor-oss/python-sdk/blob/main/examples/agentic_workflows/llm_chat.py) | 两个 LLM 之间的自动化多轮科学问答 |
| [llm_chat_human_in_loop.py](https://github.com/conductor-oss/python-sdk/blob/main/examples/agentic_workflows/llm_chat_human_in_loop.py) | 通过 WAIT 任务暂停以等待用户输入的交互式聊天 |
| [multiagent_chat.py](https://github.com/conductor-oss/python-sdk/blob/main/examples/agentic_workflows/multiagent_chat.py) | 由主持人在辩手之间路由的多智能体辩论 |
| [function_calling_example.py](https://github.com/conductor-oss/python-sdk/blob/main/examples/agentic_workflows/function_calling_example.py) | LLM 根据用户查询选择调用哪个 Python 函数 |
| [mcp_weather_agent.py](https://github.com/conductor-oss/python-sdk/blob/main/examples/agentic_workflows/mcp_weather_agent.py) | 使用 MCP 工具进行天气查询的 AI 智能体 |

**LLM 与 RAG 工作流**

| Example | Description |
|---------|-------------|
| [rag_workflow.py](https://github.com/conductor-oss/python-sdk/blob/main/examples/rag_workflow.py) | 端到端 RAG：文档转换（PDF/Word/Excel）、pgvector 索引、语义搜索、答案生成 |
| [vector_db_helloworld.py](https://github.com/conductor-oss/python-sdk/blob/main/examples/orkes/vector_db_helloworld.py) | 向量数据库操作：文本索引、嵌入生成，以及使用 Pinecone 的语义搜索 |

```shell
# Automated multi-turn chat
python examples/agentic_workflows/llm_chat.py

# Multi-agent debate
python examples/agentic_workflows/multiagent_chat.py --topic "renewable energy"

# RAG pipeline
pip install "markitdown[pdf]"
python examples/rag_workflow.py document.pdf "What are the key findings?"
```

---

## 为什么选择 Conductor？

| | |
|---|---|
| **语言无关** | 工作者可用 Python、Java、Go、JS、C# 编写——全部纳入同一个工作流 |
| **持久化执行** | 可承受崩溃，自动重试，永不丢失状态 |
| **内置 HTTP/Wait/JS 任务** | 常见操作无需编写代码 |
| **水平扩展** | 在 Netflix 构建，支撑数百万工作流 |
| **完全可视化** | UI 展示每一次执行、每一个任务、每一次重试 |
| **同步 + 异步执行** | 既可以启动后不管，也可以等待结果 |
| **人机协同** | WAIT 任务暂停直至收到外部信号 |
| **AI 原生** | 内置 LLM 聊天、RAG 管道、函数调用、MCP 工具 |

---

## 示例

完整目录见[示例指南](https://github.com/conductor-oss/python-sdk/blob/main/examples/README.md)。主要示例：

| Example | Description | Run |
|---------|-------------|-----|
| [workers_e2e.py](https://github.com/conductor-oss/python-sdk/blob/main/examples/workers_e2e.py) | 端到端：同步 + 异步工作者，指标 | `python examples/workers_e2e.py` |
| [kitchensink.py](https://github.com/conductor-oss/python-sdk/blob/main/examples/kitchensink.py) | 所有任务类型（HTTP、JS、JQ、Switch） | `python examples/kitchensink.py` |
| [workflow_ops.py](https://github.com/conductor-oss/python-sdk/blob/main/examples/workflow_ops.py) | 暂停、恢复、终止、重试、重启、重新运行、发送信号 | `python examples/workflow_ops.py` |
| [task_context_example.py](https://github.com/conductor-oss/python-sdk/blob/main/examples/task_context_example.py) | 使用 TaskInProgress 的长时间运行任务 | `python examples/task_context_example.py` |
| [metrics_example.py](https://github.com/conductor-oss/python-sdk/blob/main/examples/metrics_example.py) | Prometheus 指标收集 | `python examples/metrics_example.py` |
| [fastapi_worker_service.py](https://github.com/conductor-oss/python-sdk/blob/main/examples/fastapi_worker_service.py) | FastAPI：将工作流暴露为 API（+ 工作者） | `uvicorn examples.fastapi_worker_service:app --port 8081 --workers 1` |
| [helloworld.py](https://github.com/conductor-oss/python-sdk/blob/main/examples/helloworld/helloworld.py) | 最简 hello world | `python examples/helloworld/helloworld.py` |
| [dynamic_workflow.py](https://github.com/conductor-oss/python-sdk/blob/main/examples/dynamic_workflow.py) | 以编程方式构建工作流 | `python examples/dynamic_workflow.py` |
| [test_workflows.py](https://github.com/conductor-oss/python-sdk/blob/main/examples/test_workflows.py) | 工作流单元测试 | `python -m unittest examples.test_workflows` |

**API 旅程示例**

覆盖每个领域全部 API 的端到端示例：

| Example | APIs | Run |
|---------|------|-----|
| [authorization_journey.py](https://github.com/conductor-oss/python-sdk/blob/main/examples/authorization_journey.py) | 授权 API | `python examples/authorization_journey.py` |
| [metadata_journey.py](https://github.com/conductor-oss/python-sdk/blob/main/examples/metadata_journey.py) | 元数据 API | `python examples/metadata_journey.py` |
| [schedule_journey.py](https://github.com/conductor-oss/python-sdk/blob/main/examples/schedule_journey.py) | 调度 API | `python examples/schedule_journey.py` |
| [prompt_journey.py](https://github.com/conductor-oss/python-sdk/blob/main/examples/prompt_journey.py) | 提示（Prompt）API | `python examples/prompt_journey.py` |

## 文档

| Document | Description |
|----------|-------------|
| [工作者设计](https://github.com/conductor-oss/python-sdk/blob/main/docs/design/WORKER_DESIGN.md) | 架构：AsyncTaskRunner 与 TaskRunner、发现机制、生命周期 |
| [工作者指南](https://github.com/conductor-oss/python-sdk/blob/main/docs/WORKER.md) | 所有工作者模式（函数、类、注解、异步） |
| [工作者配置](https://github.com/conductor-oss/python-sdk/blob/main/WORKER_CONFIGURATION.md) | 分层环境变量配置 |
| [工作流管理](https://github.com/conductor-oss/python-sdk/blob/main/docs/WORKFLOW.md) | 启动、暂停、恢复、终止、重试、搜索 |
| [工作流测试](https://github.com/conductor-oss/python-sdk/blob/main/docs/WORKFLOW_TESTING.md) | 使用模拟输出的单元测试 |
| [任务管理](https://github.com/conductor-oss/python-sdk/blob/main/docs/TASK_MANAGEMENT.md) | 任务操作 |
| [元数据](https://github.com/conductor-oss/python-sdk/blob/main/docs/METADATA.md) | 任务与工作流定义 |
| [授权](https://github.com/conductor-oss/python-sdk/blob/main/docs/AUTHORIZATION.md) | 用户、群组、应用、权限 |
| [调度](https://github.com/conductor-oss/python-sdk/blob/main/docs/SCHEDULE.md) | 工作流调度 |
| [密钥](https://github.com/conductor-oss/python-sdk/blob/main/docs/SECRET_MANAGEMENT.md) | 密钥存储 |
| [提示词](https://github.com/conductor-oss/python-sdk/blob/main/docs/PROMPT.md) | AI/LLM 提示词模板 |
| [集成](https://github.com/conductor-oss/python-sdk/blob/main/docs/INTEGRATION.md) | AI/LLM 提供商集成 |
| [指标](https://github.com/conductor-oss/python-sdk/blob/main/METRICS.md) | Prometheus 指标收集 |
| [示例](https://github.com/conductor-oss/python-sdk/blob/main/examples/README.md) | 完整示例目录 |

## 支持

- 针对 SDK 的 bug、问题和功能请求，请[提交 Issue（SDK）](https://github.com/conductor-sdk/conductor-python/issues)
- 针对 Conductor OSS 服务器的问题，请[提交 Issue（Conductor server）](https://github.com/conductor-oss/conductor/issues)
- 参与社区讨论和寻求帮助，请[加入 Conductor Slack](https://join.slack.com/t/orkes-conductor/shared_invite/zt-2vdbx239s-Eacdyqya9giNLHfrCavfaA)
- 问答交流请见 [Orkes 社区论坛](https://community.orkes.io/)

## 许可证

Apache 2.0


## Examples

Browse all examples on GitHub: [conductor-oss/python-sdk/examples](https://github.com/conductor-oss/python-sdk/tree/main/examples)

| Example | Type |
|---|---|
| [Readme](https://github.com/conductor-oss/python-sdk/blob/main/examples/README.md) | file |
| [Agentic Workflow](https://github.com/conductor-oss/python-sdk/blob/main/examples/agentic_workflow.py) | file |
| [Agentic Workflows](https://github.com/conductor-oss/python-sdk/tree/main/examples/agentic_workflows) | directory |
| [Authorization Journey](https://github.com/conductor-oss/python-sdk/blob/main/examples/authorization_journey.py) | file |
| [Dynamic Workflow](https://github.com/conductor-oss/python-sdk/blob/main/examples/dynamic_workflow.py) | file |
| [Event Listener Examples](https://github.com/conductor-oss/python-sdk/blob/main/examples/event_listener_examples.py) | file |
| [Fastapi Worker Service](https://github.com/conductor-oss/python-sdk/blob/main/examples/fastapi_worker_service.py) | file |
| [Helloworld](https://github.com/conductor-oss/python-sdk/tree/main/examples/helloworld) | directory |
| [Kitchensink](https://github.com/conductor-oss/python-sdk/blob/main/examples/kitchensink.py) | file |
| [Metadata Journey](https://github.com/conductor-oss/python-sdk/blob/main/examples/metadata_journey.py) | file |
| [Metadata Journey Oss](https://github.com/conductor-oss/python-sdk/blob/main/examples/metadata_journey_oss.py) | file |
| [Metrics Example](https://github.com/conductor-oss/python-sdk/blob/main/examples/metrics_example.py) | file |
| [Orkes](https://github.com/conductor-oss/python-sdk/tree/main/examples/orkes) | directory |
| [Prompt Journey](https://github.com/conductor-oss/python-sdk/blob/main/examples/prompt_journey.py) | file |
| [Rag Workflow](https://github.com/conductor-oss/python-sdk/blob/main/examples/rag_workflow.py) | file |
| [Schedule Journey](https://github.com/conductor-oss/python-sdk/blob/main/examples/schedule_journey.py) | file |
| [Shell Worker](https://github.com/conductor-oss/python-sdk/blob/main/examples/shell_worker.py) | file |
| [Task Configure](https://github.com/conductor-oss/python-sdk/blob/main/examples/task_configure.py) | file |
| [Task Context Example](https://github.com/conductor-oss/python-sdk/blob/main/examples/task_context_example.py) | file |
| [Task Listener Example](https://github.com/conductor-oss/python-sdk/blob/main/examples/task_listener_example.py) | file |
| [Task Workers](https://github.com/conductor-oss/python-sdk/blob/main/examples/task_workers.py) | file |
| [Test Ai Examples](https://github.com/conductor-oss/python-sdk/blob/main/examples/test_ai_examples.py) | file |
| [Test Workflows](https://github.com/conductor-oss/python-sdk/blob/main/examples/test_workflows.py) | file |
| [Untrusted Host](https://github.com/conductor-oss/python-sdk/blob/main/examples/untrusted_host.py) | file |
| [User Example](https://github.com/conductor-oss/python-sdk/tree/main/examples/user_example) | directory |
| [Worker Configuration Example](https://github.com/conductor-oss/python-sdk/blob/main/examples/worker_configuration_example.py) | file |
| [Worker Discovery](https://github.com/conductor-oss/python-sdk/tree/main/examples/worker_discovery) | directory |
| [Worker Example](https://github.com/conductor-oss/python-sdk/blob/main/examples/worker_example.py) | file |
| [Workers E2E](https://github.com/conductor-oss/python-sdk/blob/main/examples/workers_e2e.py) | file |
| [Workers E2E Workflow](https://github.com/conductor-oss/python-sdk/blob/main/examples/workers_e2e_workflow.json) | file |
| [Workflow Ops](https://github.com/conductor-oss/python-sdk/blob/main/examples/workflow_ops.py) | file |
| [Workflow Status Listner](https://github.com/conductor-oss/python-sdk/blob/main/examples/workflow_status_listner.py) | file |
