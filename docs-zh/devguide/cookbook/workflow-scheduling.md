---
description: 可运行的定时计划示例集，基于标准调度器示例。
---

# 定时工作流示例集

这些示例复用了 `scheduler/examples/` 下已签入仓库的 fixture 文件。语义方面，请从[调度指南](../how-tos/Workflows/scheduling-workflows.md)看起；精确的 REST 契约参见[Scheduler API](../../documentation/api/scheduler.md)。

## 每分钟

```json
--8<-- "scheduler/examples/every-minute-schedule.json"
```

```bash
conductor schedule create scheduler/examples/every-minute-schedule.json
```

## 指定时区的工作日

```json
--8<-- "scheduler/examples/daily-report-schedule.json"
```

IANA 时区会跟随当地的夏令时切换。如果提供了关联 ID（correlation ID），它就是字面值；在工作流内部，请使用注入的 `_executionId` 来标识每一次运行。

## 补跑错过的 cron 槽位

```json
--8<-- "scheduler/examples/catchup-schedule.json"
```

补跑（catchup）可能在停机后产生一次突发。请确保目标工作流是幂等的，并且感知容量。

## 将计划限定在时间窗口内

`scheduler/examples/bounded-schedule-template.json` 包含 `__START_MS__` 和 `__END_MS__` 占位符。在提交文件之前，请把它们替换为 epoch 毫秒数；该模板本身被有意设计为不是合法的最终计划载荷。

```bash
curl -sS -X POST 'http://localhost:8080/api/scheduler/schedules' \
  -H 'Content-Type: application/json' \
  --data-binary @bounded-schedule.json
```

## 在工作流中读取调度器元数据

标准工作流使用 `_scheduledTime` 和 `_executedTime` 计算报告时间窗口：

```json
--8<-- "scheduler/examples/input-param-workflow.json"
```

与之配对的计划是：

```json
--8<-- "scheduler/examples/input-param-schedule.json"
```

其他注入的值有 `_startedByScheduler`、`_executionId` 和 `_schedulerCron`。

## 演示重叠的运行

```json
--8<-- "scheduler/examples/concurrent-schedule.json"
```

Conductor 没有原生的重叠策略。配对的 `concurrent-workflow.json` 演示了：在上一次执行仍处于活动状态时，下一个槽位依然可以启动。

## 更多标准 fixture

该 fixture 家族还包括重试、`DO_WHILE` 以及并行的多步骤工作流。在创建与之配对的计划之前，请先通过元数据 API 或 CLI 注册工作流文件。完整的本地演练步骤见[`scheduler/examples/README.md`](https://github.com/conductor-oss/conductor/blob/main/scheduler/examples/README.md)。
