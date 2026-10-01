---
description: "Conductor 生产环境最佳实践——幂等性、带指数退避的重试逻辑、超时、载荷管理、工作者水平扩展、补偿事务模式，以及面向大规模持久化执行的部署策略。"
---

# 最佳实践

本指南介绍将 Conductor 作为持久化执行引擎大规模运行的生产环境最佳实践。这里的每一条建议都来自真实世界的运维经验。


## 幂等工作者

Conductor 保证**至少一次**任务投递。网络分区、工作者重启和响应超时都可能导致同一个任务被投递多次。你的工作者必须是幂等的——同一任务执行两次应当产生相同的结果，且没有副作用。

**幂等性模式：**

| 模式 | 适用场景 |
| :--- | :--- |
| **幂等键** | 向下游服务传递一个唯一键（例如 `workflowId + taskId`）。服务基于该键去重。 |
| **使用 Upsert 而非 Insert** | 使用 `INSERT ... ON CONFLICT UPDATE` 或等价机制，使重复写入收敛到同一状态。 |
| **先检查后执行** | 在执行操作前先查询当前状态。如果已完成则跳过。 |
| **幂等 HTTP 方法** | 当下游 API 支持时，优先使用 PUT 而非 POST。 |

```python
from conductor.client.worker.worker_task import worker_task

@worker_task(task_definition_name="charge_payment")
def charge_payment(workflow_id: str, task_id: str, amount: float, currency: str) -> dict:
    idempotency_key = f"{workflow_id}-{task_id}"

    # Check if this charge was already processed
    existing = payment_gateway.get_charge(idempotency_key)
    if existing:
        return {"chargeId": existing.id, "status": "already_processed"}

    charge = payment_gateway.create_charge(
        amount=amount, currency=currency, idempotency_key=idempotency_key
    )
    return {"chargeId": charge.id, "status": "charged"}
```

`workflowId` 和 `taskId` 的组合对每次任务执行尝试都是唯一的，因此是理想的幂等键。


## 超时配置

每个任务定义都应该有显式的超时。没有超时的任务可能无限期地阻塞一个工作流。

**规则：** `responseTimeoutSeconds` < `timeoutSeconds`。响应超时用于检测无响应的工作者；总超时用于保障 SLA。

### 推荐配置

| 任务模式 | `responseTimeoutSeconds` | `timeoutSeconds` | `timeoutPolicy` | `retryCount` |
| :--- | :--- | :--- | :--- | :--- |
| API 调用（预期 < 5s） | 10 | 30 | `RETRY` | 3 |
| ML 推理 | 120 | 300 | `RETRY` | 1 |
| 人工审批 | 0（禁用） | 86400 | `ALERT_ONLY` | 0 |
| 批处理 | 600 | 3600 | `TIME_OUT_WF` | 0 |
| 快速数据转换 | 5 | 15 | `RETRY` | 3 |

### 超时策略

| 策略 | 行为 | 适用场景 |
| :--- | :--- | :--- |
| `RETRY` | 最多重试该任务 `retryCount` 次。 | 预期会出现瞬时故障（网络调用、外部 API）。 |
| `TIME_OUT_WF` | 立即使整个工作流失败。 | 任务很关键且重试无用（例如批处理窗口已过期）。 |
| `ALERT_ONLY` | 将任务标记为超时，但让工作流继续运行。 | 人工介入任务或有外部完成信号的任务。 |

!!! warning
    将 `responseTimeoutSeconds` 设置为 0 会禁用响应超时。仅对由外部完成的任务这样做（例如 [WAIT](../documentation/configuration/workflowdef/systemtasks/wait-task.md) 或 [Human](../documentation/configuration/workflowdef/systemtasks/human-task.md) 任务）。

完整的参数参考请参见 [任务定义](../documentation/configuration/taskdef.md)。


## 载荷管理

Conductor 将任务的输入和输出存储在数据库中。大载荷会降低性能并增加存储成本。

### 大小指引

| 载荷 | 推荐上限 | 硬性上限（可配置） |
| :--- | :--- | :--- |
| 任务输入 | < 64 KB | 1 MB |
| 任务输出 | < 64 KB | 1 MB |
| 工作流输入 | < 64 KB | 1 MB |

### 外部载荷存储

对于超过 64 KB 的载荷，请使用外部载荷存储。Conductor 开箱即用支持 S3：

```json
{
  "conductor.external-payload-storage.type": "s3",
  "conductor.external-payload-storage.s3.bucket-name": "my-conductor-payloads",
  "conductor.external-payload-storage.s3.region": "us-east-1",
  "conductor.external-payload-storage.s3.signed-url-expiration-seconds": 300
}
```

### 应该做与不应该做

| 应该做 | 不应该做 |
| :--- | :--- |
| 只返回下游任务需要的数据。 | 把整个 API 响应塞进任务输出。 |
| 将大文件存入 S3/GCS 并传递 URI。 | 在载荷中以 base64 传递文件内容。 |
| 使用 `inputTemplate` 在任务定义上设置默认值。 | 在每个工作流定义中重复静态配置。 |
| 保持载荷键扁平且含义明确。 | 把载荷嵌套 5 层并使用含义模糊的键。 |


## 工作流设计

### 小而专注的任务，优于单体工作者

把工作拆分成小的任务，每个任务只做一件事。这能为你带来：

- **细粒度重试** —— 只有失败的步骤会重试，而不是整条流水线。
- **可复用性** —— 小任务可以组合成不同的工作流。
- **可见性** —— 每个步骤都可以在 Conductor UI 中独立观测。

### 子工作流与内联任务

| 方式 | 适用场景 |
| :--- | :--- |
| [子工作流](../documentation/configuration/workflowdef/operators/sub-workflow-task.md) | 跨多个父工作流共享的可复用逻辑。可独立版本管理和测试。 |
| 单个工作流内的内联任务 | 只针对某一个工作流的逻辑。需要调试的间接层更少。 |

当一组任务代表一个**有边界的业务能力**（例如“处理支付”、“发送通知包”）时，使用子工作流。不要为单个任务创建子工作流——开销不值得。

### FORK_JOIN_DYNAMIC 与顺序循环

| 模式 | 适用场景 |
| :--- | :--- |
| [FORK_JOIN_DYNAMIC](../documentation/configuration/workflowdef/operators/dynamic-fork-task.md) | 并行处理 N 个项目。适用于项目相互独立且并行能提高吞吐量的场景。 |
| [DO_WHILE](../documentation/configuration/workflowdef/operators/do-while-task.md) | 顺序处理项目，适用于顺序很重要或共享资源需要串行化的场景。 |

!!! tip
    应根据下游系统的容量和配额来限制 `FORK_JOIN_DYNAMIC` 的扇出规模。当生成的分支数量会压垮该系统时，对输入进行分批处理。


## 工作者扩展

工作者是无状态的，可水平扩展。根据你的工作负载调整这些参数。

### 轮询间隔

轮询间隔控制工作者检查新任务的频率。间隔越短延迟越低；间隔越长服务器负载越低。

| 工作负载 | 推荐轮询间隔 |
| :--- | :--- |
| 低延迟（< 1s SLA） | 100-250 ms |
| 标准处理 | 500 ms - 1s |
| 后台 / 批处理 | 5-10s |

### 线程池大小

每个工作者实例运行可配置数量的轮询线程。从以下公式开始：

```
threads = (target_throughput * avg_task_duration_seconds) / num_worker_instances
```

例如，100 个任务/秒、平均执行时间 2 秒、5 个实例：`(100 * 2) / 5 = 40 threads` 每实例。

### 限流与并发

使用任务定义的设置来保护下游服务：

```json
{
  "name": "call_external_api",
  "rateLimitPerFrequency": 50,
  "rateLimitFrequencyInSeconds": 1,
  "concurrentExecLimit": 20
}
```

这将任务限制为全局每秒最多 50 次执行，且最多 20 个并发运行。

### 域隔离

使用 [任务域](../documentation/api/taskdomains.md) 将任务路由到特定的工作者池。常见用例：

- **环境隔离** —— 开发环境工作者只领取开发环境任务。
- **优先级通道** —— 将付费客户路由到专用容量。
- **区域亲和** —— 将任务路由到离数据最近的工作者。

更多细节请参见 [工作者扩展](how-tos/Workers/scaling-workers.md)。


## 错误处理模式

### 重试与终态失败

默认情况下，失败的任务会根据 `retryCount` 和 `retryLogic`（`FIXED`、`EXPONENTIAL_BACKOFF` 或 `LINEAR_BACKOFF`）进行重试。对于**不应**重试的错误，将任务状态设置为 `FAILED_WITH_TERMINAL_ERROR`：

```python
from conductor.client.http.models import TaskResult, TaskResultStatus

@worker_task(task_definition_name="validate_order")
def validate_order(order_id: str, items: list) -> TaskResult:
    if not items:
        result = TaskResult()
        result.status = TaskResultStatus.FAILED_WITH_TERMINAL_ERROR
        result.reason_for_incompletion = "Order has no items — not retryable"
        return result

    # ... validation logic
    return {"valid": True}
```

| 错误类型 | 策略 |
| :--- | :--- |
| 瞬时错误（网络超时、503） | 交给 Conductor 带退避地重试。 |
| 客户端错误（400、校验失败） | 返回 `FAILED_WITH_TERMINAL_ERROR`。 |
| 批处理中的部分失败 | 将部分结果作为输出返回；用工作流逻辑处理其余部分。 |

### 补偿与 Saga 模式

对于跨多个服务的工作流，请设计补偿任务，在后续步骤失败时撤销已完成的步骤。

**前向补偿** —— 修复问题并继续。在失败任务之后使用 [SWITCH](../documentation/configuration/workflowdef/operators/switch-task.md) 路由到恢复路径。

**后向补偿** —— 按相反顺序撤销已完成的工作。将其建模为一个独立的工作流，由 [失败工作流](how-tos/Workflows/handling-errors.md) 机制触发：

1. 主工作流在第 3 步失败。
2. Conductor 调用配置好的 `failureWorkflow`。
3. 失败工作流运行补偿任务：先撤销第 2 步，再撤销第 1 步。

!!! tip
    将补偿元数据（事务 ID、资源句柄）存储在每个任务的输出中，这样失败工作流就拥有回滚所需的一切。


## 版本管理与部署

Conductor 原生支持 [工作流版本管理](how-tos/Workflows/versioning-workflows.md)。请用它来实现安全的部署。

### 基于版本的红蓝部署

1. 部署带有你更改的工作流版本 N+1。
2. 在版本 N+1 上启动新的执行。
3. 让现有版本 N 的执行运行至完成（自然排空）。
4. 当所有版本 N 的执行都完成后，将其弃用或删除。

### 迁移运行中的执行

运行中的工作流会继续在其启动时的版本上执行。你无法把一个运行中的执行迁移到新版本。请为此做好规划：

- **短生命周期工作流** —— 等待排空。大多数会在几分钟内完成。
- **长生命周期工作流** —— 如果需要紧急修复，则终止并在新版本上重启。使用 [Terminate](../documentation/configuration/workflowdef/operators/terminate-task.md) API 并附上原因，然后重新触发。

### 安全回滚

如果版本 N+1 出现问题：

1. 停止在 N+1 上启动新的执行（将流量路由回 N）。
2. 让 N+1 的执行失败，或将它们终止。
3. 在从未被修改过的版本 N 上恢复。

由于工作者与工作流定义解耦，你可以独立于工作者部署来回滚工作流版本。


## 监控

跟踪以下指标，以维持 Conductor 的健康运行：

| 指标 | 含义 | 告警阈值 |
| :--- | :--- | :--- |
| 任务队列深度 | 未处理任务的积压量。 | 持续 5 分钟不断增长。 |
| 任务轮询次数（按任务类型） | 工作者是否正在积极轮询。 | 降为零。 |
| 工作流失败率 | 以 FAILED 状态结束的工作流百分比。 | 15 分钟窗口内 > 5%。 |
| 任务响应时间（p99） | 工作者距离响应超时的接近程度。 | > `responseTimeoutSeconds` 的 80%。 |
| 工作者线程利用率 | 工作者是否已饱和。 | 持续 10 分钟 > 90%。 |
| 外部载荷存储错误 | 阻止任务的 S3/GCS 写入失败。 | 任何非零计数。 |

内置的监控工具请参见 [工作者监控与扩展](how-tos/Workers/scaling-workers.md)。
