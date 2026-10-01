---
description: 为 Conductor 工作流创建和管理 cron 定时计划，包括时区、补偿执行、时间范围边界和注入的输入。
---

# 定时计划工作流

定时计划在每个匹配的 cron 时段创建一个新工作流执行。当由时钟掌握运行决策时使用它；当由消息掌握该决策时使用[事件编排](../event-bus.md)。

## 前置条件

- 目标工作流定义已注册。
- 服务器上的调度器已启用，且其持久化模块已配置。
- 目标工作流所需的工作者正在运行。
- 已配置支持简单 CRUD 的 Conductor CLI，或可访问提供完整调度器模型的 REST。

## 创建简单定时计划

规范示例每分钟在 UTC 下运行一次：

```json
--8<-- "scheduler/examples/every-minute-schedule.json"
```

使用 CLI 创建它：

```bash
conductor schedule create scheduler/examples/every-minute-schedule.json
conductor schedule get every-minute-demo-schedule
```

成功的标志是保存的定时计划具有非空的 `nextRunTime`，随后在下一个时段后出现工作流执行。多表达式 cron 定时计划、边界、补偿执行行为、预览和执行历史搜索请使用 REST；CLI 版本并不能一致地暴露每个调度器字段或操作。

## 使用完整的 REST 接口

```bash
curl -sS -X POST 'http://localhost:8080/api/scheduler/schedules' \
  -H 'Content-Type: application/json' \
  --data-binary @scheduler/examples/every-minute-schedule.json
```

相同的 `POST` 按定时计划名称创建或更新，并返回 `200 OK` 及存储的定时计划。确切的请求体、查询参数和状态码参见[调度器 API](../../../documentation/api/scheduler.md)。

## Cron 与时区行为

Conductor 使用 Spring 六字段 cron 表达式：秒、分、时、日、月、星期。`@daily` 等宏也被 Spring 解析器接受。

旧版单表达式形式使用 `cronExpression` 加 `zoneId`（默认 `UTC`）。多表达式形式使用 `cronSchedules`；当该数组非空时优先于旧版字段，且每个条目有自己的 `zoneId`，默认为 UTC。

```json
{
  "name": "regional-report",
  "cronSchedules": [
    {"cronExpression": "0 0 9 * * MON-FRI", "zoneId": "America/New_York"},
    {"cronExpression": "0 0 9 * * MON-FRI", "zoneId": "Europe/London"}
  ],
  "startWorkflowRequest": {
    "name": "daily_report_workflow",
    "version": 1
  }
}
```

Cron 求值遵循所选 IANA 时区，包括夏令时切换。夏令时拨快（spring-forward）期间不存在的本地时间会被 cron 引擎跳过；重复出现的本地时间按引擎的下一时刻计算处理。在夏令时边界附近测试业务敏感的定时计划。

预览端点不接受时区参数。它在 `conductor.scheduler.schedulerTimeZone`（默认 UTC）下求值，而不是定时计划的 `zoneId`，即使 `limit` 更大也最多返回五个时间。

## 补偿执行与边界限制

`runCatchupScheduleInstances: true` 会在停机后依次推进错过的 cron 时段。这可能造成突发（burst），因此工作流和依赖必须幂等且容量感知。默认 `false` 时，调度器从当前时间继续推进，而不是重放每个错过的时段。

使用 `scheduleStartTime` 和 `scheduleEndTime` 作为毫秒纪元的闭区间边界。超出时间窗口的定时计划停止产生新运行；它不会被自动删除。

## 调度器添加的输入

调度器复制 `startWorkflowRequest.input`，然后为每个执行添加以下值：

| 输入 | 含义 |
|---|---|
| `_startedByScheduler` | 定时计划名称 |
| `_scheduledTime` | 计划的 cron 时段，毫秒纪元 |
| `_executedTime` | 实际派发时间，毫秒纪元 |
| `_executionId` | 唯一的调度器执行记录 ID |
| `_schedulerCron` | 产生本次运行的 cron 表达式和时区 |

当下游系统需要每次运行的标识时，使用 `${workflow.input._executionId}`。`startWorkflowRequest.correlationId` 按字面复制；调度器**不会**对其中的 `${scheduledTime}` 或其他模板进行插值。如果每个工作流执行都需要唯一的关联 ID，请在工作流内从注入的输入推导，或通过构造该 ID 的代码启动工作流。

## 管理定时计划

```bash
conductor schedule list
conductor schedule pause every-minute-demo-schedule
conductor schedule resume every-minute-demo-schedule
conductor schedule delete every-minute-demo-schedule
```

REST 还支持过滤、搜索、暂停原因和定时执行历史：

```bash
curl 'http://localhost:8080/api/scheduler/schedules/search?paused=false&size=20'
curl 'http://localhost:8080/api/scheduler/search/executions?freeText=every-minute-demo-schedule&size=20'
```

暂停后，验证存储的 `paused` 状态，并确认 cron 时段后没有新执行出现。恢复后，确认出现新的定时执行，并检查全部五个注入字段。

## 限制

- 没有原生的重叠策略。若前一个工作流仍在运行，下一个时段可能启动另一个执行。
- 没有用于"立即运行"或手动补跑的调度器端点。临时运行时直接启动目标工作流，并显式传递预期的时间窗口。
- 预览仅限单个 cron，上限五个，使用服务器调度器时区。
- Java、Python、TypeScript 和 Go SDK 可通过生成的或底层客户端调用 REST 接口，但该仓库未在所有 SDK 中定义一致的高级调度器 API。将 REST 视为可移植的完整接口。
- `correlationId` 是字面值，不是定时计划模板。

可运行的补偿执行、边界、并发、输入、重试和多步骤变体，请使用[定时工作流配方](../../cookbook/workflow-scheduling.md)，它们复用 `scheduler/examples/`。

<a id="how-it-works"></a>
<a id="cron-expression-format"></a>
<a id="creating-a-schedule"></a>
<a id="previewing-execution-times"></a>
<a id="pausing-and-resuming"></a>
<a id="listing-and-searching-schedules"></a>
<a id="viewing-execution-history"></a>
<a id="deleting-a-schedule"></a>
<a id="passing-input-to-scheduled-workflows"></a>
<a id="configuration"></a>
