---
description: 使用 Conductor CLI、调度器 API 或 UI 让已部署的 agent 按固定节奏运行 — 无需编写代码。
---

# 调度 Agent

已部署的 agent 就是名为该 agent 的工作流，所以普通的调度器就能运行它。你不需要碰 SDK 就可以让 agent 按节奏运行 — CLI、API 和 UI 都可以。

两件事必须已经成立：

- agent 已经**部署**，因此存在一个以其名称命名的工作流。
- 有东西在**服务**它的工作者，否则被触发的执行永远不会推进。参见[部署 Agents](deploying-agents.md)。

## 使用 CLI

```bash
conductor schedule create \
  -n nightly_digest-nightly \
  -c "0 0 2 * * ?" \
  -w nightly_digest \
  -i '{"prompt":"Summarise yesterday."}'
```

| 标志 | 含义 |
|---|---|
| `-n`, `--name` | 调度名称。约定为 `{agent}-{purpose}` |
| `-c`, `--cron` | Quartz cron — **六个字段**，秒在前 |
| `-w`, `--workflow` | 已部署 agent 的名称 |
| `-i`, `--input` | 每次触发的输入，JSON 格式 |
| `-p`, `--paused` | 创建但不启动 |
| `--version` | 固定一个 agent 版本（`0` = 最新） |

你也可以从文件创建，这对发布流水线更合适：

```bash
conductor schedule create schedule.json
```

检视已存在的调度：

```bash
conductor schedule list
conductor schedule get nightly_digest-nightly
conductor schedule search -w nightly_digest      # executions the schedule produced
```

`conductor schedule list` 会打印调度、它的 cron、它启动的工作流，以及它是否处于活动状态：

```text
NAME                     CRON          WORKFLOW              STATUS   CREATED TIME
nightly_digest-nightly   0 0 2 * * ?   llm_with_guardrails   active   2026-07-27 19:48:38
```

通过 API 暂停和恢复调度：

```bash
curl -X PUT 'http://localhost:8080/api/scheduler/schedules/nightly_digest-nightly/pause'
curl -X PUT 'http://localhost:8080/api/scheduler/schedules/nightly_digest-nightly/resume'
```

## 使用 API

一切都位于 `/api/scheduler` 之下。

**创建或更新** — 同一个端点完成两者：

```bash
curl -X POST 'http://localhost:8080/api/scheduler/schedules' \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "nightly_digest-nightly",
    "cronExpression": "0 0 2 * * ?",
    "zoneId": "UTC",
    "paused": false,
    "runCatchupScheduleInstances": false,
    "description": "nightly incident digest",
    "startWorkflowRequest": {
      "name": "nightly_digest",
      "version": 1,
      "input": { "prompt": "Summarise yesterday." }
    }
  }'
```

| 字段 | 默认值 | 作用 |
|---|---|---|
| `name` | *必填* | 调度名称 |
| `cronExpression` | *必填* | 六字段的 Quartz cron |
| `startWorkflowRequest` | *必填* | 启动哪个 agent、带什么输入 |
| `zoneId` | `UTC` | 评估 cron 所用的时区 |
| `paused` | `false` | 只注册不启动 |
| `runCatchupScheduleInstances` | `false` | 补放服务器宕机期间错过的触发 |
| `scheduleStartTime` / `scheduleEndTime` | — | 调度处于生效时间的纪元边界 |
| `cronSchedules` | — | 多组 cron/时区对；优先于 `cronExpression` |
| `description` | — | 自由文本，显示在 UI 中 |

**其余操作：**

| 操作 | 调用 |
|---|---|
| 列出全部 | `GET /api/scheduler/schedules` |
| 列出某个 agent 的 | `GET /api/scheduler/schedules?workflowName=nightly_digest` |
| 获取一个 | `GET /api/scheduler/schedules/{name}` |
| 暂停 | `PUT /api/scheduler/schedules/{name}/pause` |
| 恢复 | `PUT /api/scheduler/schedules/{name}/resume` |
| 删除 | `DELETE /api/scheduler/schedules/{name}` |
| 它产生的执行 | `GET /api/scheduler/search/executions` |

**在你确定采用某个 cron 之前先校验它。** 这会返回接下来的触发时间（纪元毫秒）：

```bash
curl 'http://localhost:8080/api/scheduler/nextFewSchedules?cronExpression=0+0+2+*+*+%3F&limit=3'
# [1785290400000,1785376800000,1785463200000]
```

还有一些服务器级的管理控制 — `GET /api/scheduler/admin/pause`、`/admin/resume` 和 `/admin/requeue` — 它们会停止或重启*所有*调度。事故处理时有用，误用时危险。

## 在 UI 中

调度显示在 **[http://localhost:8080/scheduler](http://localhost:8080/scheduler)**，单个调度在 `/scheduler/edit/{name}`。UI 是暂停一个行为异常的调度、以及在不手算 cron 的情况下查看下次触发时间的最快方式。

每次被触发的运行都会像其他 agent 执行一样出现在 **[Executions](http://localhost:8080/executions)** 中。

## cron 是六个字段

Conductor 使用 Quartz cron，其中第一个字段是**秒**。五个字段的 Unix cron 不会按你预期的方式工作。

| Cron | 含义 |
|---|---|
| `0 0 2 * * ?` | 每天 02:00 |
| `0 0 * ? * *` | 每小时整点 |
| `0 */15 * ? * *` | 每 15 分钟 |
| `0 0 9 ? * MON-FRI` | 工作日 09:00 |

## 生产注意事项

- **光部署不够 — 还要服务工作者。** 一个没有任何东西服务的定时 agent 会累积永远不会推进的执行。
- **调试时暂停而不是删除**；定义和历史都会保留下来。
- **除非工作是幂等的，否则保持补放关闭。** 故障恢复后它会一次性触发所有错过的运行。
- **对任何面向业务的东西显式设置 `zoneId`。** 对用户来说，"每天凌晨 2 点"很少是指 `UTC`。
- **留意重叠。** 节奏短于 agent 运行时间的话，上一次还没结束，下一次触发就开始了。
- **调度命名为 `{agent}-{purpose}`**，这样随着数量增长，`conductor schedule list` 仍可读。

## 后续步骤

- [部署 Agents](deploying-agents.md) — 先让 agent 和它的工作者跑起来
- [Agent 配置](agent-configuration.md) — 为无人值守运行的 agent 设置边界
- [调度工作流](../how-tos/Workflows/scheduling-workflows.md) — 同一个调度器，面向普通工作流
