---
description: Conductor OSS 中 Workflow Message Queue（WMQ）功能的架构参考——通过基于 Redis 的队列向运行中的工作流推送消息、PULL_WORKFLOW_MESSAGES 系统任务，以及清理生命周期钩子。
---

# Workflow Message Queue（WMQ）— 架构

## 概述

Workflow Message Queue（WMQ）是 Conductor 的一个可选功能，允许外部系统在任意时刻向运行中的工作流推送任意 JSON 消息。工作流在新的系统任务类型 `PULL_WORKFLOW_MESSAGES` 定义的检查点处消费这些消息。

WMQ 引入了一个由 Redis 支撑的按工作流划分的消息缓冲区。外部调用方将消息推送到 REST 端点。工作流通过该任务读取消息：如果已有消息在等待，任务立即完成；否则保持 `IN_PROGRESS`，直到消息到达。

这是一个与 Conductor 现有机制不同的能力。参见下文[为什么不是现有机制](#why-not-existing-mechanisms)。


## 主要用例

| 用例 | 说明 |
|---|---|
| **Agentic / 智能体循环** | AI 智能体工作流循环等待外部调用方返回的工具结果或人工确认。调用方在工具响应时推送一条消息；循环随即解除阻塞。 |
| **Webhook 驱动的工作流** | 异步 HTTP 回调需要向已暂停的工作流输入数据。回调目标是 WMQ 推送端点，而非轮询机制。 |
| **通知管道** | 工作流循环读取可配置批次的消息，并分叉扇出到多个渠道。 |
| **人在环路（Human-in-the-loop）** | 结构化的人工决策或审批载荷由运维工具或 UI 注入到运行中的工作流。 |


## 为什么不是现有机制 { #why-not-existing-mechanisms }

| 机制 | 为何不满足 WMQ 的需求 |
|---|---|
| **事件处理器（Event handlers）** | 作用于工作流定义级别，而非每个工作流实例。它们无法按 ID 针对某个特定运行中的执行。 |
| **WAIT 任务** | 没有结构化的消息载荷。通过外部完成 API 解除阻塞时，不会有任何数据进入任务输出。 |
| **HTTP 任务** | 要求工作流主动访问某个端点。WMQ 反转了这一模型：工作流*接收*由外部方推送的数据。 |


## 架构组件

### 1. 消息队列存储

存储层以接口形式定义在 `core` 中，实现在 `redis-persistence` 中。

**接口：** `com.netflix.conductor.dao.WorkflowMessageQueueDAO`

操作：
- `push(workflowId, message)` — 向工作流的队列追加一条消息
- `pop(workflowId, maxCount)` — 从队首原子地出队最多 `maxCount` 条消息
- `size(workflowId)` — 返回当前队列深度
- `delete(workflowId)` — 删除整个队列键

**Redis 实现：** `com.netflix.conductor.redis.dao.RedisWorkflowMessageQueueDAO`

Redis 数据结构：每个工作流一个 List。

| 属性 | 详情 |
|---|---|
| 键模式 | `wmq:{workflowId}` |
| 入队 | `RPUSH` — 追加到队尾，保证 FIFO 顺序 |
| 出队 | 用 `LRANGE` 读取 + `LTRIM` 删除。因为 Conductor 在 decide 周期内持有按工作流划分的执行锁，所以即使没有原子 Lua 脚本也是安全的。对于 Redis 6.2+，`LPOP key count` 可以简化这一点。 |
| TTL | 可配置；默认 24 小时（86,400 秒）。每次 `RPUSH` 都会重置为完整 TTL。 |
| 最大容量 | 可配置上限；默认 1,000 条消息。队列达到容量时 `push` 返回错误。 |

Redis list 中存储的每条消息都是一个符合下文消息结构的 JSON 字符串。


### 2. 消息结构

每条消息都是一个具有以下字段的 JSON 对象：

```json
{
  "id": "3f2504e0-4f89-11d3-9a0c-0305e82c3301",
  "workflowId": "8e2c14e1-99ab-4c10-b4a8-a7b0d2f0e123",
  "payload": {
    "decision": "approved",
    "approvedBy": "user@example.com"
  },
  "receivedAt": "2025-06-15T10:30:00Z"
}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | UUID v4 字符串 | 在接收时由推送端点生成。返回给调用方。 |
| `workflowId` | 字符串 | 拥有该消息的工作流实例。与队列键冗余，但为便于下游追踪而包含。 |
| `payload` | 任意 JSON 对象 | 由外部调用方提供的数据。Conductor 不解释也不校验其结构。 |
| `receivedAt` | ISO-8601 UTC 时间戳 | 在接收时记录。 |


### 3. REST API

**端点：** `POST /api/workflow/{workflowId}/messages`

**请求体：** 任意 JSON 对象（即消息的 `payload` 字段）

**响应：** 以纯字符串形式返回生成的消息 `id`

**校验：**
- 目标工作流必须存在。
- 工作流必须处于 `RUNNING` 状态。向处于 `PAUSED`、`COMPLETED`、`FAILED`、`TIMED_OUT` 或 `TERMINATED` 状态的工作流推送会被拒绝，并返回相应的 HTTP 错误。

**功能开关保护：** 仅当 WMQ 功能启用时才会注册 REST 控制器 bean（参见[功能开关](#6-feature-flag)）。禁用时，端点完全不存在——它不会出现在 Swagger UI 中，并返回 404。

**推送后的副作用：** 将消息存入 Redis 后，端点会调用 `workflowExecutor.decide(workflowId)`。这会触发一次立即的工作流评估周期，使得处于进行中的 `PULL_WORKFLOW_MESSAGES` 任务无需等待下一个 SystemTaskWorker 轮询间隔即可被唤醒。参见[与 WorkflowSweeper 的交互](#interaction-with-workflowsweeper)。


### 4. PULL_WORKFLOW_MESSAGES 系统任务

`PULL_WORKFLOW_MESSAGES` 是一个异步系统任务（`isAsync() = true`）。它与 `SystemTaskWorker` 轮询循环集成。

**任务类型字符串：** `PULL_WORKFLOW_MESSAGES`

**输入参数（来自任务定义的 `inputParameters`）：**

| 参数 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `batchSize` | int | 1 | 单次调用中出队的最大消息数。受服务端 `maxBatchSize` 配置的限制。 |

**生命周期：**

1. **`start()`** — 当任务进入 `SCHEDULED` 状态时调用一次。将任务状态设为 `IN_PROGRESS`。
2. **`execute()`** — 由 `SystemTaskWorker` 在每个轮询周期调用。检查 Redis 队列。
   - 如果队列为**空**：返回 `false`。任务保持 `IN_PROGRESS`。`AsyncSystemTaskExecutor` 以较短的 `callbackAfterSeconds` 将该任务消息重新排队。
   - 如果队列**非空**：原子地弹出最多 `batchSize` 条消息，写入输出，将状态设为 `COMPLETED`，返回 `true`。

**输出字段（COMPLETED 时）：**

| 字段 | 类型 | 说明 |
|---|---|---|
| `messages` | 消息对象数组 | 已出队的消息，每条包含 `id`、`workflowId`、`payload` 和 `receivedAt`。 |
| `count` | int | 实际返回的消息数。总是 `<= batchSize`。 |

**超时行为：** 该任务遵循工作流任务定义或任务定义元数据中配置的 `timeoutSeconds` 字段。如果任务在等待消息时超时，Conductor 会应用标准的超时机制，并将任务转换为 `TIMED_OUT`。`PULL_WORKFLOW_MESSAGES` 本身无需特殊处理。


### 5. 配置属性

该功能由一个前缀为 `conductor.workflow-message-queue` 的 `@ConfigurationProperties` 类控制。

| 属性 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `conductor.workflow-message-queue.enabled` | boolean | `false` | 总开关。所有 WMQ bean 都以它为 `true` 为前提。 |
| `conductor.workflow-message-queue.maxQueueSize` | int | 1000 | 单个工作流队列中同时允许的最大消息数。 |
| `conductor.workflow-message-queue.ttlSeconds` | long | 86400 | 应用于 Redis 键的 TTL（秒）。每次推送时重置。 |
| `conductor.workflow-message-queue.maxBatchSize` | int | 100 | 任何单次 `PULL_WORKFLOW_MESSAGES` 执行中 `batchSize` 的服务端上限。 |


### 6. 功能开关 { #6-feature-flag }

```properties
conductor.workflow-message-queue.enabled=false
```

当为 `false`（默认值）时：
- 不会创建 REST 控制器 bean。
- 不会创建 `PULL_WORKFLOW_MESSAGES` 系统任务 bean。`SystemTaskRegistry` 不认识该任务类型。任何引用它的工作流定义都会校验失败。
- 不会创建 `RedisWorkflowMessageQueueDAO` bean。
- 不会创建 `WorkflowMessageQueueCleanupListener` bean。

该功能禁用时具有零运行时占用。现有部署不受影响。


### 7. 生命周期清理

队列清理依赖通过 `conductor.workflow-message-queue.ttlSeconds` 配置的 Redis TTL（默认 24 小时）。每次 `push` 都会重置 TTL，因此活动队列不会被过早过期。

没有用于清理的显式 `WorkflowStatusListener` 实现。`WorkflowStatusListener` 是单 bean 接口；添加一个 WMQ 实现会与其他监听器实现（例如归档监听器）冲突。Redis TTL 已经足够：任何孤立的队列（例如工作流完成前服务器崩溃之后留下的）都会自动过期。


## 数据流

### 推送流

```
External caller
  └─ POST /api/workflow/{wfId}/messages  (JSON payload)
       └─ WorkflowMessageQueueResource
            ├─ Validate workflow exists and is RUNNING
            ├─ Generate message ID (UUID v4)
            ├─ Serialize message to JSON
            ├─ dao.push(workflowId, message)
            │    └─ RPUSH wmq:{workflowId} <json>
            │    └─ EXPIRE wmq:{workflowId} <ttlSeconds>
            ├─ workflowExecutor.decide(workflowId)   ← triggers immediate re-evaluation
            └─ Return message ID to caller
```

### 拉取流 — 消息已在等待（正常路径）

```
WorkflowExecutor schedules PULL_WORKFLOW_MESSAGES task
  └─ task status: SCHEDULED
       └─ SystemTaskWorker.execute()
            └─ PullWorkflowMessages.start()
                 └─ task status: IN_PROGRESS
                      └─ AsyncSystemTaskExecutor calls PullWorkflowMessages.execute()
                           └─ dao.pop(workflowId, batchSize) → [message1, ...]
                                └─ task output: { messages: [...], count: N }
                                └─ task status: COMPLETED
                                     └─ WorkflowExecutor.decide() advances workflow
```

### 拉取流 — 等待消息（尚无消息）

```
PullWorkflowMessages.execute()
  └─ dao.pop(workflowId, batchSize) → []   (queue empty)
       └─ return false
            └─ AsyncSystemTaskExecutor re-queues task with callbackAfterSeconds
                 └─ SystemTaskWorker polls again at next interval
                      └─ [repeat until message arrives]

[Meanwhile, external caller pushes a message via POST /api/workflow/{wfId}/messages]
  └─ workflowExecutor.decide(workflowId) called after push
       └─ AsyncSystemTaskExecutor.execute() triggered
            └─ PullWorkflowMessages.execute()
                 └─ dao.pop() → [message]
                      └─ task COMPLETED, workflow advances
```

### 清理

队列键通过 Redis TTL（默认 24 小时，每次推送时重置）自动过期。工作流完成时不会触发任何显式的清理钩子。


## 与 WorkflowSweeper 的交互 { #interaction-with-workflowsweeper }

`PULL_WORKFLOW_MESSAGES` 是一个异步系统任务（`isAsync() = true`）。这意味着：

1. 当工作流引擎调度该任务时，它会被放入系统任务队列（一个以任务类型为键、由 `QueueDAO` 支撑的队列）。
2. `SystemTaskWorkerCoordinator` 在启动时将该任务注册到 `SystemTaskWorker`，后者开始轮询其队列。
3. 每次轮询都会调用 `AsyncSystemTaskExecutor.execute()`。它首次执行时调用 `start()`，之后的调用使用 `execute()`。
4. 当 `execute()` 返回 `false`（队列为空）时，`AsyncSystemTaskExecutor` 会以 `systemTaskCallbackTime` 秒（通过 `conductor.app.systemTaskWorkerCallbackDuration` 配置）的延迟将任务 ID 重新推回系统任务队列。
5. 当通过 REST API 推送消息时，会立即调用 `workflowExecutor.decide(workflowId)`。Sweeper 重新评估工作流，并对任何 `IN_PROGRESS` 的异步任务触发 `AsyncSystemTaskExecutor`。这把唤醒延迟从等待完整的轮询间隔降低到接近实时。

`PULL_WORKFLOW_MESSAGES` 重写了 `getEvaluationOffset()` 并返回 `Optional.of(1L)`，因此任务在等待消息期间每 1 秒重新评估一次，而不是默认的 `systemTaskWorkerCallbackDuration`（30 秒）。


## 失败模式与弹性

| 场景 | 行为 |
|---|---|
| **推送时 Redis 不可用** | `dao.push()` 抛出异常。REST 端点返回 HTTP 500。消息不会被保存。调用方必须重试。由于没有任何内容被持久化，所以不会丢失消息。 |
| **拉取时 Redis 不可用** | `dao.pop()` 抛出异常。`PullWorkflowMessages.execute()` 传播错误。`AsyncSystemTaskExecutor` 按照标准的重试/超时配置处理任务级失败。任务可能会被重试或超时。 |
| **PULL_WORKFLOW_MESSAGES 处于 IN_PROGRESS 时工作流终止** | WorkflowSweeper 检测到终止状态并取消待处理任务。`WorkflowMessageQueueCleanupListener` 删除队列键。 |
| **PULL_WORKFLOW_MESSAGES 处于 IN_PROGRESS 时工作流被暂停** | 任务保持 `IN_PROGRESS`。暂停期间推送的消息会积压在 Redis 中（受 `maxQueueSize` 限制）。工作流恢复时，会调用 `workflowExecutor.decide()`，sweeper 重新评估，任务在下一次轮询时取走积压的消息。 |
| **队列超出大小限制** | 当队列达到 `maxQueueSize` 时，`dao.push()` 返回错误。REST 端点返回 HTTP 429（Too Many Requests）或 HTTP 400。调用方必须处理背压。 |
| **非常大的消息载荷** | 载荷以行内方式存储在 Redis list 条目中。对于非常大的数据，请使用外部存储引用模式：将数据存储在对象存储（S3、GCS 等）中，只在 WMQ 载荷中放入引用 URL 或键。这与 Conductor 的外部载荷存储使用相同的模式。 |
| **重复投递** | Lua 出队脚本在单个 Redis 实例内是原子的，因此同一条消息不会在一次 `execute()` 调用中被投递两次。不过，系统层面适用至少一次（at-least-once）语义——调用方和工作流设计者应尽可能将已消费的消息视为幂等的。 |
| **Conductor 节点之间发生网络分区** | 如果运行着多个 Conductor 实例，Redis List 是共享的。原子 Lua 出队保证同一工作流两个并发的 `execute()` 调用不会返回重叠的消息。工作流级锁（`ExecutionLockService`）在 decide 路径上提供额外保护。 |


## 安全与访问控制注意事项

- 推送端点接收任意 JSON。Conductor 不校验载荷结构。实现方应在 API 网关层添加认证/授权。
- 推送 URL 中的 `workflowId` 参数足以定位任何运行中的工作流。调用方必须是可信的，或者该端点必须受到保护。
- 载荷数据以配置的 TTL 存储在 Redis 中。载荷中的敏感数据受 Redis 访问控制约束。如有需要，考虑在应用层对敏感字段加密。


## 组件汇总

| 组件 | 位置 | 用途 |
|---|---|---|
| `WorkflowMessageQueueDAO` | `core` | 定义存储契约的接口 |
| `WorkflowMessage` | `common` | 表示单条消息的 POJO |
| `WorkflowMessageQueueProperties` | `core` | 所有 WMQ 设置的 `@ConfigurationProperties` |
| `PullWorkflowMessages` | `core`（系统任务） | 将消息出队到工作流输出的系统任务 |
| `RedisWorkflowMessageQueueDAO` | `redis-persistence` | DAO 的 Redis List 实现 |
| `WorkflowMessageQueueConfiguration` | `core` | 装配 InMemory DAO（默认回退方案）的 Spring `@Configuration` |
| `RedisWorkflowMessageQueueConfiguration` | `redis-persistence` | 在 Redis 激活时装配 Redis DAO 的 Spring `@Configuration` |
| `WorkflowMessageQueueResource` | `rest` | 推送端点的 REST 控制器 |
