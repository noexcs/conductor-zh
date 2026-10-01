---
description: "Conductor 中的任务定义 Schema——为持久化工作流执行配置重试逻辑、指数退避、超时、限流与并发。"
---

# 任务定义

完整的机器可读字段契约，请参阅 [TaskDef.json](schemas.md#definition-objects)。

任务定义（Task Definition）用于注册 SIMPLE 任务（工作者）。Conductor 维护着一个用户任务类型的注册表。任务类型在使用前必须先注册。

不要将其与 [*任务配置*](workflowdef/index.md#task-configurations) 混淆，后者是工作流定义的一部分，在定义的 `tasks` 属性中逐条遍历。


## Schema

| 字段                       | 类型               | 描述                                                                                                                                                                                                                      | 说明                                                                            |
| :-------------------------- | :----------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | :------------------------------------------------------------------------------- |
| name                        | string             | 任务名称。与任务功能相符的唯一名称。                                                                                                                                                                                       | 必须唯一                                                                   |
| description                 | string             | 任务的描述。                                                                                                                                                                                       | 可选                                                                         |
| retryCount                  | number             | 任务被标记为失败时尝试的重试次数。                                                                                                                                                 | 默认 3，最大允许值以 10 为上限                                  |
| retryLogic                  | string (enum)      | 重试机制。                                                                                                                                                                                     | 见 [重试逻辑](#retry-logic)                                                  |
| retryDelaySeconds           | number             | 首次重试前的基础延迟。含义因 `retryLogic` 而异。                                                                                                                                        | 默认 60 秒                                                           |
| maxRetryDelaySeconds        | number             | 重试之间的最大延迟（秒）。为 `EXPONENTIAL_BACKOFF` 和 `LINEAR_BACKOFF` 计算出的延迟设置上限，使延迟不会超过该值。`0` 表示不启用上限。                              | 默认 0（无上限）。见 [重试逻辑](#retry-logic)                          |
| backoffJitterMs             | number             | 为每次重试延迟添加最多该毫秒数的随机抖动，把同时发生的重试在时间上分散开，防止惊群效应（thundering herd）。`0` 表示不启用抖动。                                          | 默认 0（无抖动）。见 [重试逻辑](#retry-logic)                       |
| totalTimeoutSeconds         | number             | 所有重试尝试合计的最大墙上时钟时间（秒）。一旦超过，无论 `retryCount` 是否用完，任务都会立即失败且不再重试。`0` 表示不启用该限制。             | 默认 0（无限制）。见 [超时场景](../../devguide/architecture/tasklifecycle.md#total-timeout)  |
| timeoutPolicy               | string (enum)      | 任务的超时策略。                                                                                                                                                                                         | 默认 `TIME_OUT_WF`；见 [超时策略](#timeout-policy)                 |
| timeoutSeconds              | number             | 任务首次进入 `IN_PROGRESS` 状态后，若超过该秒数仍未到达终态，则被标记为 `TIMED_OUT`。                                                                | 设为 0 则不启用超时                                                          |
| responseTimeoutSeconds      | number             | 若大于 0，则任务在此时长后若未更新状态将被重新调度（心跳机制）。适用于工作者已轮询到任务，却因错误/网络故障无法完成的情况。 | 默认 600                                                                 |
| pollTimeoutSeconds          | number             | 若超过该秒数仍未被工作者轮询，任务将被标记为 `TIMED_OUT`。                                                                                                                     | 设为 0 则不启用超时                                                          |
| inputKeys                   | array of string(s) | 任务预期输入的键数组。用于记录任务的输入。                                                                                                                                    | 可选。见 [使用 inputKeys 与 outputKeys](#using-inputkeys-and-outputkeys)。 |
| outputKeys                  | array of string(s) | 任务预期输出的键数组。用于记录任务的输出。                                                                                                                                  | 可选。见 [使用 inputKeys 与 outputKeys](#using-inputkeys-and-outputkeys)。 |
| inputTemplate               | object             | 定义默认输入值。                                                                                                                                                                                  | 可选。见 [使用 inputTemplate](#using-inputtemplate)                        |
| concurrentExecLimit         | number             | 任意给定时刻可执行的任务数。                                                                                                                                                        | 可选                                                                         |
| rateLimitFrequencyInSeconds | number             | 设置限流的频率窗口。                                                                                                                                                                         | 可选。见 [任务限流](#task-rate-limits)                              |
| rateLimitPerFrequency       | number             | 设置窗口内可派发给工作者的最大任务数。                                                                                                                                      | 可选。见下文 [任务限流](#task-rate-limits)                        |
| ownerEmail                  | string             | 负责该任务的团队邮箱地址。                                                                                                                                                                  | 必填                                                                         |

### 重试逻辑 { #retry-logic }

`retryLogic` 字段控制重试之间延迟的计算方式。实际应用的最终延迟为：

```
delay = clamp(computedDelay, 0, maxRetryDelaySeconds)  +  random(0, backoffJitterMs) ms
```

其中 `clamp` 仅在 `maxRetryDelaySeconds > 0` 时生效。

| 取值 | 延迟公式 | 说明 |
| :--- | :--- | :--- |
| `FIXED` | `retryDelaySeconds` | 每次重试均为固定延迟。 |
| `EXPONENTIAL_BACKOFF` | `retryDelaySeconds × 2^attemptNumber` | 每次尝试翻倍。用 `maxRetryDelaySeconds` 设置上限，避免延迟失控增长。 |
| `LINEAR_BACKOFF` | `retryDelaySeconds × backoffScaleFactor × attemptNumber` | 线性增长。`backoffScaleFactor` 默认为 1。 |

**`maxRetryDelaySeconds`** — 对计算出的延迟设置上限，使其永不超过该值。示例：`EXPONENTIAL_BACKOFF`、`retryDelaySeconds=1`、`maxRetryDelaySeconds=3`：

| 尝试 | 原始延迟 | 应用上限后 |
| :--- | :--- | :--- |
| 0 | 1s | 1s |
| 1 | 2s | 2s |
| 2 | 4s | 3s |
| 3+ | 8s+ | 3s |

**`backoffJitterMs`** — 在最终延迟上叠加 `[0, backoffJitterMs]` 毫秒区间内均匀分布的随机值。这能将多个失败工作者的重试在时间上分散开（防止惊群效应）。示例：`retryDelaySeconds=2`、`backoffJitterMs=1000` → 每次重试在失败后 2000 ms 至 3000 ms 之间触发。

### 超时策略 { #timeout-policy }

* `RETRY`: 重新重试该任务
* `TIME_OUT_WF`: 工作流被标记为 TIMED_OUT 并终止。这是默认值。
* `ALERT_ONLY`: 注册一个计数器（task_timeout）

### 任务并发执行限制

`concurrentExecLimit` 限制任意时刻同时执行的任务数。

**示例**
假设有 1000 个任务执行在队列中等待，同时有 1000 个工作者在轮询该队列获取任务；但如果 `concurrentExecLimit` 设置为 10，则只有 10 个任务会被派发给工作者（这会导致饥饿）。每当有工作者完成执行，就会从队列中取出新任务，同时始终保持当前执行数为 10。

### 任务限流 { #task-rate-limits }

* `rateLimitFrequencyInSeconds` 和 `rateLimitPerFrequency` 应配合使用。
* `rateLimitFrequencyInSeconds` 设置"频率窗口"，即 `events per duration` 中使用的 `duration`。例如：1s、5s、60s、300s 等。
* `rateLimitPerFrequency` 定义每个"频率窗口"内可派发给工作者的任务数。设为 0 则不启用限流。

**示例**
假设设置 `rateLimitFrequencyInSeconds = 5`、`rateLimitPerFrequency = 12`。这意味着频率窗口为 5 秒，且每个频率窗口内 Conductor 只会派发 12 个任务给工作者。因此，在任意一分钟内，无论有多少工作者在轮询任务，Conductor 都只会派发 12*(60/5) = 144 个任务给工作者。

注意，与 `concurrentExecLimit` 不同，限流不考虑已在执行中或已到达终态的任务。即使之前的任务都在 1 秒内执行完，或者需要几天才能完成，新任务仍会按配置的频率派发给工作者，上例即每分钟 144 个任务。


### 使用 `inputKeys` 与 `outputKeys` { #using-inputkeys-and-outputkeys }

* `inputKeys` 和 `outputKeys` 可以视为任务的参数和返回值。
* 可以把任务定义想象成一个接口：```(value1, value2 .. valueN) someTaskDefinition(key1, key2 .. keyN);```。
* 不过，目前这些参数并不被严格强制。`inputKeys` 和 `outputKeys` 都作为任务复用的文档而存在。工作流中的任务无需定义任务定义中的所有键。
* 未来，这可以扩展为一种严格的模板，所有任务实现都必须遵循，就像编程语言中的接口一样。

### 使用 `inputTemplate` { #using-inputtemplate }

* `inputTemplate` 允许定义默认值，这些值可被工作流中提供的值覆盖。
* 例如：在你的任务定义中，可以这样定义 inputTemplate：

```json
"inputTemplate": {
    "url": "https://some_url:7004"
}
```

* 现在，在你的工作流定义中使用上面的任务时，可以使用默认的 `url`，也可以在任务的 `inputParameters` 中用其他值覆盖它。

```json
"inputParameters": {
    "url": "${workflow.input.some_new_url}"
}
```

## 重试配置示例

### 重试不稳定的外部 API 调用

```json
{
  "name": "call_payment_api",
  "retryCount": 5,
  "retryLogic": "EXPONENTIAL_BACKOFF",
  "retryDelaySeconds": 2,
  "maxRetryDelaySeconds": 60,
  "backoffJitterMs": 2000,
  "responseTimeoutSeconds": 30,
  "timeoutSeconds": 300,
  "timeoutPolicy": "RETRY",
  "ownerEmail": "payments@example.com"
}
```

最多重试 5 次，延迟依次为 2s、4s、8s、16s、32s——以 60s 为上限——且每次尝试附加最多 2 秒的随机抖动。可避免对状态不佳的支付服务商反复冲击。

### 使用 `totalTimeoutSeconds` 限制重试预算

```json
{
  "name": "process_order",
  "retryCount": 10,
  "retryLogic": "FIXED",
  "retryDelaySeconds": 5,
  "totalTimeoutSeconds": 120,
  "timeoutPolicy": "TIME_OUT_WF",
  "ownerEmail": "orders@example.com"
}
```

每 5 秒重试一次，但整个序列——所有尝试合计——必须在 2 分钟内完成。即使 `retryCount` 尚未用完，一旦 2 分钟的预算耗尽，任务即失败。

### 带抖动的高吞吐工作者

```json
{
  "name": "send_notification",
  "retryCount": 3,
  "retryLogic": "FIXED",
  "retryDelaySeconds": 1,
  "backoffJitterMs": 3000,
  "concurrentExecLimit": 500,
  "ownerEmail": "notifications@example.com"
}
```

当数千个通知同时失败时（例如下游服务故障），抖动会把重试分散到一个 3 秒的窗口内，而不是让所有重试同时冲击该服务。

## 完整示例

``` json
{
  "name": "encode_task",
  "retryCount": 3,
  "retryLogic": "EXPONENTIAL_BACKOFF",
  "retryDelaySeconds": 10,
  "maxRetryDelaySeconds": 120,
  "backoffJitterMs": 5000,
  "totalTimeoutSeconds": 600,
  "timeoutSeconds": 1200,
  "timeoutPolicy": "TIME_OUT_WF",
  "responseTimeoutSeconds": 3600,
  "pollTimeoutSeconds": 3600,
  "inputKeys": [
    "sourceRequestId",
    "qcElementType"
  ],
  "outputKeys": [
    "state",
    "skipped",
    "result"
  ],
  "concurrentExecLimit": 100,
  "rateLimitFrequencyInSeconds": 60,
  "rateLimitPerFrequency": 50,
  "ownerEmail": "foo@bar.com",
  "description": "Sample Encoding task"
}
```
