---
description: "Conductor 上 AI agent 的精确失败契约 — 当 LLM 调用失败、工具超时、人类不响应、回调重复到达、分支部分完成、版本在执行中途变更、worker 在活动执行期间部署时，究竟会发生什么。"
---

# AI agent 的失败语义

本页精确定义 agent 工作流中出问题时发生的事情。不是笼统地说"Conductor 是持久化的"，而是 agent 可能遇到的每一种失败场景下的精确行为。


## LLM 任务失败

**场景：** `LLM_CHAT_COMPLETE` 任务调用 LLM 提供方，而调用失败了（限流、超时、提供方故障、响应格式错误）。

**会发生什么：**

1. 任务进入 `FAILED`。
2. Conductor 检查任务的重试配置（`retryCount`、`retryLogic`、`retryDelaySeconds`）。
3. 创建一个重试次数加一的新任务执行。
4. 任务在经过配置的延迟后重新入队。
5. 如果所有重试用尽，任务进入 `FAILED` 终止状态。
6. 工作流的失败处理生效：如果配置了 `failureWorkflow` 就运行它，否则工作流进入 `FAILED`。

**什么会被保留：** prompt、错误响应、重试次数，以及每次尝试的时间信息。你可以在 UI 中检视每一次失败的尝试。

**什么不会被重新执行：** 上游的任何东西都不会。只有失败的 LLM 调用会被重试。之前已完成的任务保留其输出。

**配置：**

```json
{
  "name": "plan_action",
  "retryCount": 3,
  "retryLogic": "EXPONENTIAL_BACKOFF",
  "retryDelaySeconds": 5,
  "responseTimeoutSeconds": 60
}
```

这会让 LLM 调用按指数退避最多重试 3 次（5s、10s、20s）。如果 LLM 在 60 秒内没有响应，任务就会超时并重试。


## LLM 返回格式错误的输出

**场景：** LLM 有响应，但输出不是有效的 JSON，或不匹配预期的 schema（例如缺少 `action` 字段）。

**会发生什么：**

`LLM_CHAT_COMPLETE` 任务成功完成 — LLM 确实响应了。格式错误的输出会传播到下一个任务。接下来发生什么取决于下游任务：

- 如果下一个任务引用 `${plan.output.result.action}` 而 `action` 不存在，该任务会以输入解析错误失败。
- 该任务按其重试策略重试。
- LLM **不会**被再次调用（它已经完成了）。

**如何处理：** 在 LLM 调用之后添加一个 `SWITCH` 或 `INLINE` 任务，在处理输出之前先校验它：

```json
{
  "name": "validate_plan",
  "taskReferenceName": "validate",
  "type": "INLINE",
  "inputParameters": {
    "plan": "${plan.output.result}",
    "evaluatorType": "graaljs",
    "expression": "(function() { var p = $.plan; if (!p || !p.action) { return {valid: false, error: 'Missing action field'}; } return {valid: true, plan: p}; })()"
  }
}
```

如果校验失败，使用 `SWITCH` 带上纠正性的 prompt 重新运行 LLM，或者直接让工作流失败。


## 工具调用超时

**场景：** `CALL_MCP_TOOL` 或 `HTTP` 任务调用一个外部工具，而工具没有在配置的超时时间内响应。

**会发生什么：**

1. `responseTimeoutSeconds` 到期。任务进入 `TIMED_OUT`。
2. 如果配置了重试，任务会被重试。一个新请求会发送到工具。
3. 原始（已超时的）请求可能仍在途。工具最终可能会处理它。

**关键影响：** 工具调用可能执行超过一次。**工具工作者和 MCP 工具应当是幂等的。** 使用任务的 `taskId` 或一个关联 ID 作为幂等键。

**什么会被保留：** 超时的尝试会连同其输入、超时事件和时间信息一起被记录。每次重试尝试都会单独记录。


## 工具调用在产生副作用后失败

**场景：** 一个工具调用发送了一封邮件，然后 worker 在报告完成之前崩溃。任务被重试，邮件被再次发送。

**会发生什么：**

1. worker 轮询到任务、开始执行，并发出了邮件。
2. worker 在调用 `POST /api/tasks` 报告完成之前崩溃。
3. `responseTimeoutSeconds` 到期。任务进入 `TIMED_OUT`，然后是 `SCHEDULED`（重试）。
4. 新的 worker 接管任务并再次发送邮件。

**这是至少一次投递（at-least-once delivery）。** Conductor 保证任务至少执行一次，但如果 worker 在执行副作用之后失败，任务可能执行多次。

**如何处理：**

- 让有副作用的操作幂等。使用幂等键（`taskId` 对每次尝试都是唯一的）。
- 使用任务的 `updateTime` 来检测重复投递 — 如果任务已被处理过，就跳过该副作用。
- 对于不可逆的副作用，配置带补偿任务的 `failureWorkflow`。


## 人类始终不响应

**场景：** 一个 `HUMAN` 任务在等待审批，但没有人响应。几小时过去了。几天过去了。

**会发生什么：**

`HUMAN` 任务会无限期地以 `IN_PROGRESS` 状态保存在持久化存储中。除非你在任务定义上显式配置了 `timeoutSeconds`，否则它不会超时。

- 等待期间工作流不消耗任何计算资源。没有轮询、没有定时器、没有线程。
- 任务可以熬过服务器重启、部署和基础设施变更。
- 任务在 UI 中可见，并可通过 API 查询。

**如果你想要超时：** 在任务定义上设置 `timeoutSeconds` 和 `timeoutPolicy`：

```json
{
  "name": "human_approval",
  "timeoutSeconds": 86400,
  "timeoutPolicy": "TIME_OUT_WF"
}
```

这会在 24 小时后超时并让工作流失败。或者使用 `timeoutPolicy: "ALERT_ONLY"` 只记录超时而不让工作流失败。

**如果你想要升级处理（escalation）：** 使用并行的 `WAIT` + `HUMAN` 模式：

```json
{
  "type": "FORK",
  "forkTasks": [
    [{"type": "HUMAN", "taskReferenceName": "approval"}],
    [{"type": "WAIT", "inputParameters": {"duration": "4 hours"}},
     {"type": "LLM_CHAT_COMPLETE", "taskReferenceName": "escalation_notify"}]
  ]
}
```


## 回调被投递了两次

**场景：** 外部系统调用 Task Update API 来完成一个 `HUMAN` 任务，但网络不稳定，调用被重试了。Conductor 收到了两次完成信号。

**会发生什么：**

第一次调用把任务从 `IN_PROGRESS` 移到 `COMPLETED`，并推进工作流。第二次调用到达时，任务已经处于终止状态。

- Conductor 拒绝这次更新。任务已经是 `COMPLETED`。
- 不会发生重复执行。工作流不会推进两次。
- 第二次调用返回一个错误，指示任务已处于终止状态。

**这是默认安全的。** Conductor 的任务状态机保证一个任务只能进入终止状态一次。重复的回调是无害的。


## FORK/JOIN 中分支部分完成

**场景：** 一个 `FORK/JOIN` 运行三个并行分支。分支 1 完成。分支 2 失败。分支 3 仍在运行。

**会发生什么：**

1. 分支 2 失败。其任务进入 `FAILED`，并按其重试策略重试。
2. 分支 3 继续独立执行。
3. `JOIN` 任务等待所有分支到达终止状态。
4. 如果分支 2 用尽重试并进入终止 `FAILED`，`JOIN` 任务失败。
5. 分支 3 可能仍在运行 — 它不会被自动取消（除非工作流被终止）。
6. 工作流的失败处理生效。

**什么会被保留：** 每个分支中已完成的任务保留其输出。如果你从失败的任务重试工作流，只有失败的分支会重新执行。成功的分支不会被重跑。


## 工作流定义在执行中途变更

**场景：** 在有执行在运行时，你更新了工作流定义（添加一个任务、修改一个参数）。

**会发生什么：**

正在运行的执行**不受影响**。每个执行使用开始时刻对定义所拍的不可变快照。快照嵌入在执行记录中。

- 新的执行使用更新后的定义。
- 正在运行的执行继续按原有定义执行。
- 你可以有多个版本同时运行。

**如果你想应用新定义：** 使用[使用最新定义重启](../../architecture/durable-execution.md#replay-and-recovery)。这会使用更新后的定义从头重新执行工作流。


## 活动执行期间 worker 部署

**场景：** 你部署了 worker 代码的新版本。旧 worker 实例被停止，新实例启动。任务正在执行中。

**会发生什么：**

1. 旧 worker 被停止。它们正在处理的任务被遗弃。
2. 被遗弃任务的 `responseTimeoutSeconds` 到期。任务进入 `TIMED_OUT`，然后是 `SCHEDULED`（重试）。
3. 新的 worker 实例轮询任务并接管重新入队的任务。
4. 执行继续。

**脆弱窗口：** 旧 worker 停止到 `responseTimeoutSeconds` 到期之间的这段时间。在这个窗口内，任务看起来是 `IN_PROGRESS`，但没有任何 worker 在处理它。

**如何把影响降到最低：**

- 让 `responseTimeoutSeconds` 保持较短（大多数任务 10-60 秒）。
- 在你的 worker 中使用优雅停机 — 在停止之前完成进行中的任务。
- 对于 Conductor 服务器本身：sweeper 服务在启动时重新评估进行中的工作流，并把停滞的任务重新入队。

**什么永远不会丢失：** 已完成任务的输出。工作流状态。执行历史。只有进行中的任务会受到影响，而且它会被自动重试。


## 动态任务类型不再存在

**场景：** 一个 `DYNAMIC` 任务根据 LLM 输出解析出任务类型。LLM 返回了一个不存在的任务名（未注册、已被删除、或拼写错误）。

**会发生什么：**

`DYNAMIC` 任务以解析错误失败 — 找不到指定的任务类型。任务进入 `FAILED` 并按其重试策略重试。

**如何处理：** 在 `DYNAMIC` 任务之前校验 LLM 输出。使用 `INLINE` 或 `SWITCH` 任务检查解析出的任务名是否在已知的白名单中。


## worker 与服务器之间网络分区

**场景：** worker 正在执行一个任务（例如一个 LLM 调用）。发生网络分区。worker 完成了任务，但无法把结果报告给 Conductor 服务器。

**会发生什么：**

1. worker 完成 LLM 调用并收到响应。
2. worker 尝试向服务器报告 `COMPLETED`。由于网络分区，请求失败。
3. worker 重试状态更新（SDK 层重试）。
4. 如果分区持续时间超过 `responseTimeoutSeconds`，服务器把任务标记为 `TIMED_OUT` 并重新入队。
5. 分区恢复后，某个 worker（可能是同一个）接管任务并重新执行 LLM 调用。

**在这个场景中 token 会被消耗两次。** 原始 LLM 调用成功了，但结果丢失了。这就是至少一次投递的代价。对于长时间运行或昂贵的 LLM 调用，考虑在你的 worker 中实现客户端缓存以避免重新执行。


## 跨小时/天/周的长时运行 agent 循环

**场景：** 一个自主 agent 循环运行数小时或数天，期间有 `WAIT` 暂停、`HUMAN` 审批和周期性 LLM 调用。

**会发生什么：**

这是 Conductor 的正常运行模式。工作流保持 `RUNNING`，单个任务处于 `IN_PROGRESS`（活跃工作）或 `COMPLETED`（已完成的步骤）。

- `WAIT` 任务不消耗资源。持久化定时器在持续时间到达时触发，即使跨越部署也是如此。
- `HUMAN` 任务不消耗资源。它们会一直持久化，直到信号到达。
- `DO_WHILE` 循环计数器和中间状态会存活下来，除非工作流通过 `keepLastN` 选择进行迭代清理。
- 服务器重启、worker 部署和基础设施变更不会影响该执行。

**实际限制：**

- 执行数据随已完成任务数量线性增长。对于非常长的循环（数千次迭代），考虑把大负载卸载到外部存储，只在任务输出中存储指针。`keepLastN` 可以从任务输出和存储中删除较早的循环迭代；仅当可以接受丢失较早的历史时使用它。参见[外部负载存储](../../documentation/advanced/externalpayloadstorage.md)。
- 工作流级别的 `timeoutSeconds` 作用于整个执行。把它设置得足够高以覆盖你预期的持续时间，或者省略它以不限执行时间。


## 总结：失败契约

| 失败 | Conductor 做什么 | 你应该做什么 |
|---------|--------------------|--------------------|
| LLM 调用失败 | 按配置的退避重试 | 在任务定义上设置重试策略 |
| LLM 返回错误输出 | 下游任务在输入解析时失败 | 在 LLM 调用之后添加校验步骤 |
| 工具调用超时 | `responseTimeoutSeconds` 之后重试 | 让工具幂等 |
| 工具调用产生副作用后崩溃 | 重试 — 副作用可能执行两次 | 使用幂等键 |
| 人类始终不响应 | 任务永远保持 `IN_PROGRESS` | 设置 `timeoutSeconds` 或构建升级处理 |
| 重复回调 | 第二次调用被拒绝，无重复执行 | 默认安全 |
| FORK 分支失败 | JOIN 等待所有分支；若分支用尽重试则工作流失败 | 为每个分支配置重试策略 |
| 运行中定义变更 | 运行中的执行不受影响（快照） | 使用 restart 应用新定义 |
| Worker 部署 | 在途任务在响应超时后重新入队 | 保持响应超时较短；使用优雅停机 |
| 动态任务不存在 | 任务失败，重试 | 在 DYNAMIC 解析之前校验 LLM 输出 |
| 网络分区 | 超时后任务重新入队，可能重新执行 | 让 worker 幂等；考虑客户端缓存 |
| 多天执行 | 正常运行，完全持久化 | 卸载大负载；设置合适的超时 |

## 重试自适应循环

如果循环体中的任务失败，外层的 `DO_WHILE` 就会失败。重试那个失败的 `DO_WHILE` 会从第 1 次迭代重新开始循环的迭代历史；它不同于那种只是保留之前循环迭代的任务级重试。在工作流保持活跃期间发生的基础设施恢复会保留已持久化的状态，而普通失败任务的重试会保留已完成的上游任务。设计长生命周期的自适应循环时，请使用幂等工具、显式的迭代上限，以及足够的保留上下文，让重启是安全的。

参见 **[持久化自适应图](dynamic-workflows.md)** 了解受治理的循环模式及其 `keepLastN` 权衡。


## 后续步骤

- **[生产级 Agent 架构](production-agent-architecture.md)** — 把这个失败契约变成部署与恢复实践。
- **[生产级 Agent 架构](production-agent-architecture.md)** — 标准的端到端 agent 模式。
- **[持久化执行语义](../../architecture/durable-execution.md)** — 完整的持久化模型、任务状态机与重试配置。
- **[为什么 Agent 工作流选 Conductor](why-conductor.md)** — Conductor 为 agent 工作流开箱即用地提供什么。
- **[Token 效率](token-efficiency.md)** — 持久化执行如何在所有这些失败场景中节省 token。
