---
description: "以代码为工作流——使用 Conductor SDK 在 Python 中动态构建代码优先的工作流。条件分支、循环、并行执行以及运行时生成的动态工作流。"
---

# 以代码编写的动态工作流

## 以代码为工作流

Conductor 支持代码优先的工作流方式——使用 Python SDK 以编程方式构建工作流，而不是手写 JSON。这种"以代码为工作流"的模式让你可以用 `>>` 运算符链接任务、添加条件逻辑、循环和并行分支——全部在 Python 中完成。代码优先的工作流最适合任务图在运行时才确定的动态工作流。

### 简单顺序工作流

使用 `>>` 运算符链接任务。用 `@worker_task` 装饰的工作者函数成为可复用的任务构建块。

```python
from conductor.client.workflow.conductor_workflow import ConductorWorkflow
from conductor.client.worker.worker_task import worker_task


@worker_task(task_definition_name='fetch_order')
def fetch_order(order_id: str) -> dict:
    return {'order_id': order_id, 'amount': 99.99, 'item': 'Widget'}


@worker_task(task_definition_name='process_payment')
def process_payment(order_id: str, amount: float) -> dict:
    return {'transaction_id': 'txn_abc123', 'status': 'charged'}


@worker_task(task_definition_name='ship_order')
def ship_order(order_id: str, transaction_id: str) -> dict:
    return {'tracking': 'TRACK-456', 'carrier': 'FedEx'}


workflow = ConductorWorkflow(name='order_fulfillment', version=1, executor=executor)

fetch = fetch_order(task_ref_name='fetch', order_id=workflow.input('order_id'))
pay = process_payment(
    task_ref_name='pay',
    order_id=workflow.input('order_id'),
    amount=fetch.output('amount'),
)
ship = ship_order(
    task_ref_name='ship',
    order_id=workflow.input('order_id'),
    transaction_id=pay.output('transaction_id'),
)

workflow >> fetch >> pay >> ship
workflow.output_parameters({
    'tracking': ship.output('tracking'),
    'transaction_id': pay.output('transaction_id'),
})
workflow.register(overwrite=True)
```

---

### 使用 Switch 的条件分支

根据任务输出或工作流输入路由执行。每个 case 拥有自己的任务链。

```python
from conductor.client.workflow.conductor_workflow import ConductorWorkflow
from conductor.client.workflow.task.switch_task import SwitchTask


workflow = ConductorWorkflow(name='route_by_priority', version=1, executor=executor)

classify = classify_ticket(
    task_ref_name='classify',
    description=workflow.input('description'),
)

switch = SwitchTask(task_ref_name='priority_router', case_expression=classify.output('priority'))

# Each case is a list of tasks to execute
switch.switch_case('critical', [
    page_oncall(task_ref_name='page', ticket_id=workflow.input('ticket_id')),
    escalate(task_ref_name='escalate', ticket_id=workflow.input('ticket_id')),
])
switch.switch_case('high', [
    assign_senior(task_ref_name='assign', ticket_id=workflow.input('ticket_id')),
])
switch.default_case([
    add_to_backlog(task_ref_name='backlog', ticket_id=workflow.input('ticket_id')),
])

workflow >> classify >> switch
workflow.register(overwrite=True)
```

---

### 使用 Fork/Join 的并行执行

并行运行独立任务，并等待全部完成。

```python
from conductor.client.workflow.conductor_workflow import ConductorWorkflow
from conductor.client.workflow.task.fork_task import ForkTask
from conductor.client.workflow.task.join_task import JoinTask


workflow = ConductorWorkflow(name='parallel_enrichment', version=1, executor=executor)

# Define independent tasks
credit_check = check_credit(task_ref_name='credit', customer_id=workflow.input('customer_id'))
fraud_check = check_fraud(task_ref_name='fraud', customer_id=workflow.input('customer_id'))
kyc_check = check_kyc(task_ref_name='kyc', customer_id=workflow.input('customer_id'))

# Fork runs all branches in parallel
fork = ForkTask(
    task_ref_name='parallel_checks',
    forked_tasks=[
        [credit_check],
        [fraud_check],
        [kyc_check],
    ],
)

# Join waits for all branches
join = JoinTask(task_ref_name='wait_all', join_on=['credit', 'fraud', 'kyc'])

# Merge results
decide = make_decision(
    task_ref_name='decide',
    credit_score=credit_check.output('score'),
    fraud_risk=fraud_check.output('risk_level'),
    kyc_status=kyc_check.output('status'),
)

workflow >> fork >> join >> decide
workflow.output_parameters({'decision': decide.output('result')})
workflow.register(overwrite=True)
```

---

### 使用 Do/While 的循环

重复执行一组任务，直到满足条件——适用于轮询、重试或迭代式 AI 智能体循环。

```python
from conductor.client.workflow.conductor_workflow import ConductorWorkflow
from conductor.client.workflow.task.do_while_task import DoWhileTask


workflow = ConductorWorkflow(name='agent_loop', version=1, executor=executor)

# The task(s) to repeat each iteration
think = call_llm(
    task_ref_name='think',
    prompt=workflow.input('goal'),
)
act = execute_tool(
    task_ref_name='act',
    tool=think.output('tool'),
    args=think.output('args'),
)

# Loop until the LLM says it's done (max 10 iterations)
loop = DoWhileTask(
    task_ref_name='agent_loop',
    termination_condition='if ($.act["output"]["done"] == true) { false; } else { true; }',
    tasks=[think, act],
)
loop.input_parameters.update({'max_iterations': 10})

summarize = summarize_results(task_ref_name='summarize', results=act.output('results'))

workflow >> loop >> summarize
workflow.register(overwrite=True)
```

---

### HTTP、系统任务与工作者混用

将内置系统任务（HTTP、Wait、JQ Transform）与自定义工作者组合——系统任务无需额外部署。

{% raw %}
```python
from conductor.client.workflow.conductor_workflow import ConductorWorkflow
from conductor.client.workflow.task.http_task import HttpTask
from conductor.client.workflow.task.json_jq_task import JsonJQTask
from conductor.client.workflow.task.wait_task import WaitTask


workflow = ConductorWorkflow(name='data_pipeline', version=1, executor=executor)

# HTTP task — fetch data from an external API (no worker needed)
fetch = HttpTask(task_ref_name='fetch_data', http_input={
    'uri': 'https://api.example.com/records',
    'method': 'GET',
    'headers': {'Authorization': ['Bearer ${workflow.input.api_key}']},
})

# JQ Transform — reshape the response (no worker needed)
transform = JsonJQTask(
    task_ref_name='transform',
    script='.body.records | map({id: .id, value: .metrics.total})',
)
transform.input_parameters.update({
    'records': fetch.output('response.body'),
})

# Custom worker — run business logic
enrich = enrich_records(
    task_ref_name='enrich',
    records=transform.output('result'),
)

# Wait — pause for 5 seconds before the next step
cooldown = WaitTask(task_ref_name='cooldown', wait_for_seconds=5)

# Custom worker — store results
store = save_to_database(task_ref_name='store', records=enrich.output('enriched'))

workflow >> fetch >> transform >> enrich >> cooldown >> store
workflow.output_parameters({'stored': store.output('count')})
workflow.register(overwrite=True)
```
{% endraw %}

---

### 子工作流

将大型工作流拆分为可复用的部分。父工作流将子工作流作为任务调用。

```python
from conductor.client.workflow.conductor_workflow import ConductorWorkflow
from conductor.client.workflow.task.sub_workflow_task import SubWorkflowTask


# Child workflow (registered separately)
child = ConductorWorkflow(name='process_single_item', version=1, executor=executor)
validate = validate_item(task_ref_name='validate', item=child.input('item'))
transform = transform_item(task_ref_name='transform', item=validate.output('validated'))
child >> validate >> transform
child.output_parameters({'result': transform.output('transformed')})
child.register(overwrite=True)


# Parent workflow invokes the child
parent = ConductorWorkflow(name='batch_processor', version=1, executor=executor)

prepare = prepare_batch(task_ref_name='prepare', batch_id=parent.input('batch_id'))

run_child = SubWorkflowTask(
    task_ref_name='process_item',
    workflow_name='process_single_item',
    version=1,
)
run_child.input_parameters.update({'item': prepare.output('first_item')})

aggregate = aggregate_results(
    task_ref_name='aggregate',
    result=run_child.output('result'),
)

parent >> prepare >> run_child >> aggregate
parent.register(overwrite=True)
```

---

### 运行时生成的动态工作流

在运行时构建工作流定义并直接执行，无需预先注册。这种运行时工作流模式支持任务图即时生成的动态工作流——适用于 AI 智能体、数据流水线以及任何步骤无法预先知晓的场景。

{% raw %}
```python
from conductor.client.configuration.configuration import Configuration
from conductor.client.orkes_clients import OrkesClients
from conductor.client.http.models import StartWorkflowRequest


config = Configuration()
clients = OrkesClients(configuration=config)
executor = clients.get_workflow_executor()

# Build the workflow definition dynamically
steps = ['validate', 'enrich', 'store']  # determined at runtime

tasks = []
for i, step in enumerate(steps):
    tasks.append({
        'name': step,
        'taskReferenceName': f'{step}_{i}',
        'type': 'SIMPLE',
        'inputParameters': {
            'data': '${workflow.input.data}' if i == 0 else f'${{{steps[i-1]}_{i-1}.output.result}}',
        },
    })

# Start with inline definition — no pre-registration needed
request = StartWorkflowRequest(
    name='dynamic_pipeline',
    workflow_def={
        'name': 'dynamic_pipeline',
        'version': 1,
        'tasks': tasks,
        'outputParameters': {
            'result': f'${{{steps[-1]}_{len(steps)-1}.output.result}}',
        },
    },
    input={'data': {'key': 'value'}},
)

workflow_id = executor.start_workflow(request)
print(f'Started dynamic workflow: {workflow_id}')
```
{% endraw %}

该模式对运行时生成执行计划的 AI 智能体非常强大——LLM 产出步骤列表，你的代码构建工作流定义，Conductor 以完整的持久化、重试和可观测性执行它。

---

### 执行并等待结果

同步运行工作流并内联获取结果——适用于 API 和交互式应用。

```python
from conductor.client.configuration.configuration import Configuration
from conductor.client.orkes_clients import OrkesClients

config = Configuration()
clients = OrkesClients(configuration=config)
executor = clients.get_workflow_executor()

# Execute synchronously — blocks until the workflow completes
run = executor.execute(
    name='order_fulfillment',
    version=1,
    workflow_input={'order_id': 'ORD-789'},
)

print(f'Status:  {run.status}')
print(f'Output:  {run.output}')
print(f'View:    {config.ui_host}/execution/{run.workflow_id}')
```

---

## 环境配置

以上所有示例都假定存在一个 `WorkflowExecutor` 实例。以下是标准配置：

```python
from conductor.client.configuration.configuration import Configuration
from conductor.client.orkes_clients import OrkesClients

config = Configuration()  # reads CONDUCTOR_SERVER_URL from env
clients = OrkesClients(configuration=config)
executor = clients.get_workflow_executor()
```

```shell
pip install conductor-python
export CONDUCTOR_SERVER_URL=http://localhost:8080/api
```

要查看更多 Python SDK 示例，请参阅 [Python SDK 文档](../../documentation/clientsdks/python-sdk.md) 和 [GitHub 上的示例](https://github.com/conductor-oss/python-sdk/tree/main/examples)。
