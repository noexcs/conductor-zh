---
description: "监控任务队列并扩展 Conductor 工作者——队列深度、轮询数据、Prometheus 指标、自动扩缩策略和性能调优。"
---

# 扩展任务工作者

工作者在 Conductor 服务器之外执行业务逻辑。让它们保持健康需要两件事：**监控**队列和工作者状态，以及根据数据告诉你的信息来**扩缩容**。


## 监控任务队列

Conductor 跟踪每个任务类型的队列大小和工作者轮询活动。使用这些数据来检测积压、停滞的工作者和容量问题。

### 使用 UI

导航到 **Home > Task Queues**（或 `<your UI server URL>/taskQueue`）。对于每个任务，UI 显示：

- **Queue Size** — 等待被领取的任务。
- **Workers** — 正在轮询此任务的工作者数量和实例详情。

### 使用 CLI

```bash
# List all tasks with queue info
conductor task list

# Get details for a specific task
conductor task get <TASK_NAME>
```

### 使用 API

获取队列中等待的任务数量：

```shell
curl '{{ server_host }}{{ api_prefix }}/tasks/queue/sizes?taskType=<TASK_NAME>' \
  -H 'accept: */*'
```

获取工作者轮询数据（哪些工作者在轮询、上次轮询时间）：

```shell
curl '{{ server_host }}{{ api_prefix }}/tasks/queue/polldata?taskType=<TASK_NAME>' \
  -H 'accept: */*'
```

!!! note
    将 `<TASK_NAME>` 替换为你的任务名。


## Prometheus 指标

Conductor 发布用于仪表盘、告警和自动扩缩策略的指标。所有指标都包含 `taskType` 标签，以便按任务监控。

### 队列深度（Gauge）

```promql
max(task_queue_depth{taskType="my_task"})
```

- 保持队列深度稳定。它不需要为零（尤其是长时间运行的任务），但持续增长意味着工作者跟不上。
- 对队列深度在持续时段内上升设置告警，并用它触发自动扩缩。

### 任务完成速率（Counter）

```promql
rate(task_completed_seconds_count{taskType="my_task"}[$__rate_interval])
```

- 衡量吞吐量——每秒完成的任务数。
- 突然下降表明工作者吃力、失败或已停止轮询。
- 设置最低吞吐量阈值，当低于该值时告警。

### 队列等待时间

```promql
max(task_queue_wait_time_seconds{quantile="0.99", taskType="my_task"})
```

任务在队列中停留多久才会被工作者领取。如果这个时间超过几秒：

1. **检查工作者数量** — 如果所有工作者都忙，就增加实例。
2. **检查轮询间隔** — 如果工作者轮询不够频繁，就缩短它。

!!! warning
    缩短轮询间隔会增加对服务器的 API 请求。要在响应性和服务器负载之间取得平衡。


## 扩缩容策略

### 何时扩缩容

| 信号 | 动作 |
|---|---|
| 队列深度稳定增长 | 增加工作者实例 |
| p99 队列等待时间 > 5s | 增加工作者实例或缩短轮询间隔 |
| 队列增长的同时吞吐量下降 | 排查工作者健康状况（CPU、内存、下游依赖） |
| 队列持续为空，工作者空闲 | 缩减以节省资源 |

### 水平扩展

增加工作者实例。Conductor 自动分发任务——每个轮询同一任务类型的工作者都在同一个队列中竞争工作。Conductor 服务器无需配置变更。

### 轮询间隔调优

轮询间隔控制工作者检查新任务的频率。间隔越短，延迟越低，但服务器负载越高。

| 场景 | 推荐间隔 |
|---|---|
| 延迟敏感任务 | 100–500ms |
| 标准处理 | 1–5s |
| 批处理 / 后台工作 | 5–30s |

### 线程池大小

每个工作者实例可以运行多个轮询线程。一个好的起点：

```
threads = (task_throughput × avg_task_duration) / num_worker_instances
```

对于 I/O 密集型任务（HTTP 调用、数据库查询），使用比 CPU 核心数更多的线程。对于 CPU 密集型任务，线程数与可用核心数匹配。

### 速率限制

如果下游服务有速率限制，配置任务级速率限制，防止工作者把它们压垮：

```json
{
  "name": "call_external_api",
  "rateLimitPerFrequency": 100,
  "rateLimitFrequencyInSeconds": 60
}
```

这将限制该任务在所有工作者范围内每 60 秒窗口内执行 100 次。

### 域隔离

使用 [task-to-domain](../../../documentation/api/taskdomains.md) 把任务路由到特定的工作者池。这可以防止"吵闹的邻居"——高流量工作流不会让服务于延迟敏感工作流的工作者饿死。
