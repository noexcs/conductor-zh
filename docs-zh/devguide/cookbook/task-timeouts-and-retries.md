---
description: "Conductor 手册（cookbook）— 任务超时与重试示例集，涵盖带租约续期的 responseTimeout、totalTimeoutSeconds、带上限与抖动的指数退避，以及惊群效应预防。"
---

# 任务超时与重试

让 worker 具备韧性（resilient）的实用示例集。每个示例都是一个完整的任务定义，可以通过 `POST /api/metadata/taskdefs` 注册。

---

### 带上限的指数退避

针对调用外部 API 的任务，使用指数退避进行重试。上限防止延迟无限增长；抖动（jitter）防止多个失败的 worker 在同一时刻轰炸该 API。

```json
{
  "name": "call_payment_api",
  "ownerEmail": "payments@example.com",
  "retryCount": 6,
  "retryLogic": "EXPONENTIAL_BACKOFF",
  "retryDelaySeconds": 2,
  "maxRetryDelaySeconds": 60,
  "backoffJitterMs": 3000,
  "responseTimeoutSeconds": 30,
  "timeoutSeconds": 600,
  "timeoutPolicy": "RETRY"
}
```

**延迟时间表**（`retryDelaySeconds=2`、`maxRetryDelaySeconds=60`、`backoffJitterMs=3000`）：

| 尝试 | 基础延迟 | 上限之后 | 实际范围 |
| :--- | :--- | :--- | :--- |
| 1 | 2s | 2s | 2.0 – 5.0s |
| 2 | 4s | 4s | 4.0 – 7.0s |
| 3 | 8s | 8s | 8.0 – 11.0s |
| 4 | 16s | 16s | 16.0 – 19.0s |
| 5 | 32s | 32s | 32.0 – 35.0s |
| 6 | 64s | **60s** | 60.0 – 63.0s |

---

### 长时运行 worker 的租约续期

`responseTimeoutSeconds` 是心跳窗口：如果 worker 在该时长内没有回报，Conductor 会将任务标记为 `TIMED_OUT` 并重试它。对于耗时超过心跳窗口的任务，worker 通过提交带 `callbackAfterSeconds` 的 `IN_PROGRESS` 更新来续租。

**任务定义**

```json
{
  "name": "transcode_video",
  "ownerEmail": "media@example.com",
  "retryCount": 2,
  "retryLogic": "FIXED",
  "retryDelaySeconds": 10,
  "responseTimeoutSeconds": 30,
  "timeoutSeconds": 3600,
  "timeoutPolicy": "RETRY"
}
```

`responseTimeoutSeconds: 30` —— 如果 worker 30 秒内没有消息，Conductor 将重新调度该任务。
`timeoutSeconds: 3600` —— 任务本身跨越所有心跳的总耗时最长可达 1 小时。

**Worker：每 25 秒续租一次**

```python
import time
from conductor.client.http.models import TaskResult

def transcode_video(task):
    task_id = task.task_id
    workflow_id = task.workflow_instance_id

    for chunk in video_chunks(task.input_data["file_url"]):
        transcode_chunk(chunk)

        # Extend the lease before responseTimeoutSeconds (30s) expires.
        # callbackAfterSeconds tells Conductor to leave this task invisible
        # in the queue for another 25s — resetting the response clock.
        heartbeat = TaskResult(
            task_id=task_id,
            workflow_instance_id=workflow_id,
            status="IN_PROGRESS",
            callback_after_seconds=25,
            output_data={"progress": chunk.index / len(video_chunks)}
        )
        conductor_client.update_task(heartbeat)

    return TaskResult(
        task_id=task_id,
        workflow_instance_id=workflow_id,
        status="COMPLETED",
        output_data={"output_url": upload_result.url}
    )
```

**没有心跳时会发生什么：**

```
t=0s   Worker polls task → IN_PROGRESS
t=30s  responseTimeoutSeconds expires → TIMED_OUT → retry scheduled
t=40s  Worker finishes (too late, task already terminated)
```

**每 25 秒一次心跳时会发生什么：**

```
t=0s   Worker polls task → IN_PROGRESS
t=25s  Worker: POST IN_PROGRESS, callbackAfterSeconds=25 → clock resets
t=50s  Worker: POST IN_PROGRESS, callbackAfterSeconds=25 → clock resets
...
t=90s  Worker: POST COMPLETED → task done
```

---

### 使用 `totalTimeoutSeconds` 的硬性 SLA

当你需要为任务在全部重试中可能耗费的总时长设置一个有保障的上限时，使用 `totalTimeoutSeconds`。它与 `retryCount` 相互独立——先触发的限制优先生效。

```json
{
  "name": "sync_crm_record",
  "ownerEmail": "crm@example.com",
  "retryCount": 20,
  "retryLogic": "FIXED",
  "retryDelaySeconds": 5,
  "totalTimeoutSeconds": 120,
  "responseTimeoutSeconds": 15,
  "timeoutPolicy": "TIME_OUT_WF"
}
```

`retryCount: 20` —— 正常情况下允许 20 次重试。
`totalTimeoutSeconds: 120` —— 但如果 2 分钟的墙钟预算先被耗尽，则不再入队新的重试，工作流被标记为失败。

对于 SLA 敏感的任务很有用：无论遇到何种瞬时故障，你都能确定工作流要么成功，要么在有限的时间窗口内以失败状态呈现。

**时间线示例**（`retryDelaySeconds=5`、`totalTimeoutSeconds=30`）：

```
t=0s   Attempt 1 → FAILED
t=5s   Attempt 2 → FAILED
t=10s  Attempt 3 → FAILED
t=15s  Attempt 4 → FAILED
t=20s  Attempt 5 → FAILED
t=25s  Attempt 6 → FAILED
t=30s  totalTimeoutSeconds exceeded → workflow FAILED, no more retries
        (10 retries still remained in retryCount)
```

---

### 惊群效应预防

当数百个任务同时失败时（例如下游服务宕机），所有重试会被调度在同一时刻。没有抖动的情况下，它们会同时冲击正在恢复的服务。`backoffJitterMs` 会把它们分散到一个时间窗口内。

```json
{
  "name": "send_webhook",
  "ownerEmail": "platform@example.com",
  "retryCount": 5,
  "retryLogic": "EXPONENTIAL_BACKOFF",
  "retryDelaySeconds": 1,
  "maxRetryDelaySeconds": 30,
  "backoffJitterMs": 5000,
  "responseTimeoutSeconds": 10,
  "concurrentExecLimit": 200
}
```

设置 `backoffJitterMs: 5000` 后，在 `t=0` 同时失败的 500 个任务将在 `t=1s` 到 `t=6s` 之间的均匀随机时刻重试——把重试负载分摊到 5 秒内，而不是以单次突发冲击服务。

---

### 选择合适的组合

| 场景 | 推荐配置 |
| :--- | :--- |
| 有限速的外部 API | `EXPONENTIAL_BACKOFF` + `maxRetryDelaySeconds` + `backoffJitterMs` |
| 长时运行的处理作业 | `responseTimeoutSeconds`（短）+ worker 心跳 + `timeoutSeconds`（长） |
| 受 SLA 约束的任务 | `totalTimeoutSeconds` + `FIXED` 或 `EXPONENTIAL_BACKOFF` |
| 高扇出且大量并发失败 | `backoffJitterMs` + `concurrentExecLimit` |
| 不可重试的错误 | 从 worker 返回 `FAILED_WITH_TERMINAL_ERROR` |

所有可用参数参见[任务定义参考](../../documentation/configuration/taskdef.md)。
