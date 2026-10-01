---
description: "Conductor 手册（cookbook）— 用单个 HTTP_POLL 任务轮询缓慢的第三方作业直至完成：以 terminationCondition、pollingInterval、pollingStrategy 和 maxPollCount 取代 DO_WHILE 循环。"
---

# 轮询长时间运行的外部作业

你向第三方 API 提交一项作业，它返回一个作业 id。这个作业要运行几分钟，有时甚至几小时。你需要工作流等待它完成——不占用线程、不需要 worker，也不要不停地轰炸供应商。

`HTTP_POLL` 就是这样一个任务。你给它一个状态 URL 和一个条件——“当条件成立时停止”。

## 整体结构

```text
submit_job (HTTP)  ──>  await_job (HTTP_POLL)  ──>  SUCCEEDED ──> record artifact
                            │  polls the status URL          FAILED ──> TERMINATE
                            │  until terminationCondition
                            └─ sleeps between polls, holds nothing open
```

## 为什么不用循环

用 `DO_WHILE` 包裹一个 `HTTP` 任务也能工作，在较早的示例中你会见到这种写法。它的代价比看起来更大：

| | `DO_WHILE` + `HTTP` | `HTTP_POLL` |
|---|---|---|
| 执行中的任务数 | 每轮两个，无限增长 | 一个 |
| 轮询间的退避 | 你自己实现 | `pollingStrategy` |
| 轮询上限 | 你自己计数迭代 | `maxPollCount` |
| 查看执行 | 滚动翻过 40 轮迭代 | 一个带轮询计数的任务 |

循环版本还让*真正有意思*的部分——终止条件——变成埋在 `loopCondition` 里的一处表达式，针对循环状态而非响应求值。

## 任务

```json
{
  "name": "await_job",
  "taskReferenceName": "await_job",
  "type": "HTTP_POLL",
  "inputParameters": {
    "http_request": {
      "uri": "${workflow.input.jobApiUrl}/jobs/${submit_job.output.response.body.jobId}",
      "method": "GET",
      "terminationCondition": "(function(){ var s = $.output.response.body.state; return s === 'SUCCEEDED' || s === 'FAILED'; })();",
      "pollingInterval": 60,
      "pollingStrategy": "FIXED",
      "maxPollCount": 60
    }
  }
}
```

`HTTP_POLL` 接收与 `HTTP` 相同的 `http_request` 块——`uri`、`method`、`headers`、`body`、`accept`、`contentType`、`connectionTimeOut`、`readTimeOut`、`acceptedStatusCodes`、`outputFilter`——外加四个轮询字段：

| 字段 | 默认值 | 作用 |
|---|---|---|
| `terminationCondition` | — | 每次轮询后求值的表达式。取真值（truthy）时停止任务 |
| `pollingInterval` | — | 两次轮询之间的秒数 |
| `pollingStrategy` | — | `FIXED`、`LINEAR_BACKOFF` 或 `EXPONENTIAL_BACKOFF` |
| `maxPollCount` | `1000` | 轮询达到该次数后放弃 |

### 编写终止条件

表达式能看到两个对象：

- **`$.output`** —— 当前这次轮询的结果，包括 `response.body`、`response.headers`、`response.statusCode`
- **`$.input`** —— 任务的输入

返回布尔值来表示“完成”或“继续”。也可以返回数字做三态控制：`1` 完成任务，`0` 再次轮询，`-1` 使任务失败。

**失败时也要终止。** 只匹配 `SUCCEEDED` 的条件会一直轮询一个已死的作业，直到 `maxPollCount` 用尽。请匹配每一个终态，然后再根据结果分支：

```javascript
(function(){ var s = $.output.response.body.state; return s === 'SUCCEEDED' || s === 'FAILED'; })();
```

### 轮询间隔有服务端下限

`pollingInterval` 会被钳制到 `conductor.worker.http_poll.min_poll_interval`，其默认值为 **60 秒**。除非运维人员调低了该下限，否则请求 `pollingInterval: 5` 实际得到的仍是 60。请按生效的间隔（而不是你请求的间隔）来设置 `maxPollCount`：60 秒一次、轮询 60 次，上限就是一小时。

## 前置条件

一个正在运行的 Conductor 服务器，以及一个可供轮询的作业 API。附带了一个桩服务（stub），你可以无需供应商账号即可运行这套结构。

将其保存为 `job_stub_service.py` 并让它保持运行：

```python
--8<-- "docs/devguide/cookbook/assets/job_stub_service.py"
```

```bash
python3 job_stub_service.py       # http://localhost:8089
```

它每次轮询推进一个状态——`QUEUED` → `RUNNING` → `RUNNING` → `SUCCEEDED`——这样你无需等待真实墙钟时间就能观察完整生命周期。`POST /jobs/{id}/fail` 强制触发失败分支，`GET /polls` 显示每个作业被轮询了多少次。

## 可运行的定义

将其保存为 `http-poll-external-job.json`：

```json
--8<-- "docs/devguide/cookbook/assets/http-poll-external-job.json"
```

## 注册并运行

```bash
conductor workflow create http-poll-external-job.json
conductor workflow start -w http_poll_external_job \
  -i '{"jobApiUrl":"http://localhost:8089","dataset":"orders_2026_q2"}'
```

在 Conductor UI 中打开 **[执行（Executions）](http://localhost:8080/executions)**，选择新的执行，查看任务图以及每个任务的输入和输出。

`await_job` 始终保持为单个任务，其轮询计数不断攀升。当桩服务报告 `SUCCEEDED` 时，`SWITCH` 记录该工件（artifact）；用 `POST /jobs/{id}/fail` 强制失败，则同一个工作流改为以 `remote_job_failed` 终止。

交叉核对供应商实际看到的内容：

```bash
curl -s http://localhost:8089/polls
```

## 生产环境注意事项

- **在条件中匹配每一个终态，**而不仅仅是成功，否则已死的作业会一直被轮询到 `maxPollCount` 用尽。
- **`pollingInterval` 有服务端下限**（`min_poll_interval`，默认 60 秒）。你的值只是一个请求，不是保证。
- **按墙钟预算设置 `maxPollCount`。** 间隔 × 次数才是实际上限；给工作流设置一个高于它的 `timeoutSeconds`。
- **对时长未知的作业使用 `EXPONENTIAL_BACKOFF`，**以免一个五小时的作业产生 300 个相同的请求。
- **轮询廉价的端点。** 如果供应商的状态调用有限流或返回完整载荷，请向对方要一个轻量的状态 URL，或使用 `outputFilter` 避免把响应保留进工作流状态。
- **提交步骤需要幂等键。** 如果重试的提交又创建了第二个作业，你轮询的就不是目标作业了。
- **不要用它处理亚秒级的工作。** 低于轮询下限时，同步 `HTTP` 任务才是合适的工具。

## 相关内容

- [等待与定时器模式](wait-and-timers.md) — 等待信号或时钟，而不是状态 URL
- [任务超时与重试](task-timeouts-and-retries.md) — 为提交调用设置界限
- [Saga：补偿部分失败](saga-compensation.md) — 当后续步骤失败时撤销已提交的作业
