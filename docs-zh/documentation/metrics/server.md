---
description: "服务器指标 — 使用基于 Micrometer 的指标和告警来监控 Conductor 服务器的健康状态和性能。"
---
# 服务器指标

!!! Info "功能更新"
    自 [v3.21.16](https://github.com/conductor-oss/conductor/releases/tag/v3.21.16) 起，Conductor 已改用 [Micrometer](https://micrometer.io/) 进行指标采集。


Conductor 使用 [Micrometer](https://micrometer.io/) 进行指标的采集与导出。

以下指标由 Conductor 服务器发布。你可以导出这些指标，为你的工作流和任务设置告警。

| 指标名称          | 描述       | 标签  |
| ------------- |:----------------- | ----- |
| workflow_server_error | 服务器端错误发生的速率。  | methodName|
| workflow_failure | 失败的工作流数量。                           |workflowName, status|
| workflow_start_error | 启动失败的工作流数量。           |workflowName|
| workflow_running | 正在运行的工作流数量。                          | workflowName, version|
| workflow_execution | 工作流完成所花费的时间。                 | workflowName, ownerApp |
| task_queue_wait | 任务在队列中花费的时间。               | taskType |
| task_execution | 执行一个任务所花费的时间。                           | taskType, includeRetries, status |
| task_poll | 轮询一个任务所花费的时间。                               | taskType|
| task_poll_count | 任务被轮询的次数。              | taskType, domain |
| task_queue_depth | 待处理任务的队列深度。                        | taskType, ownerApp |
| task_rate_limited | 当前被限流的任务数量。 | taskType |
| task_concurrent_execution_limited | 当前受并发执行限制约束的任务数量。 | taskType |
| task_timeout | 已超时的任务数量。 | taskType |
| task_response_timeout | 因 `responseTimeout` 超时的任务数量。 | taskType |
| task_update_conflict | 任务更新冲突的数量。 <br/><br/> 例如，即使工作流已进入终止状态，工作者仍在更新任务状态。 | workflowName, taskType, taskStatus, workflowStatus |
| event_queue_messages_processed | 从事件队列中获取的消息数量。 | queueType, queueName |
| observable_queue_error | 从事件队列获取消息时遇到的错误数量。 | queueType |
| event_queue_messages_handled | 从事件队列中取出并执行的消息数量。 | queueType, queueName |
| external_payload_storage_usage | 使用外部 payload 存储的次数。 | name, operation, payloadType |


## 支持的监控系统

Conductor 支持以下 Micrometer 指标实现：

- [Atlas](https://docs.micrometer.io/micrometer/reference/implementations/atlas.html)
- [Prometheus](https://docs.micrometer.io/micrometer/reference/implementations/prometheus.html)
- [Datadog](https://docs.micrometer.io/micrometer/reference/implementations/datadog.html)
- [JMX](https://docs.micrometer.io/micrometer/reference/implementations/jmx.html)
- [OpenTelemetry Protocol (OTPL)](https://docs.micrometer.io/micrometer/reference/implementations/otlp.html)
- [Dynatrace](https://docs.micrometer.io/micrometer/reference/implementations/dynatrace.html)
- [Elasticsearch](https://docs.micrometer.io/micrometer/reference/implementations/elastic.html)
- [New Relic](https://docs.micrometer.io/micrometer/reference/implementations/new-relic.html)
- [StackDriver](https://docs.micrometer.io/micrometer/reference/implementations/stackdriver.html)
- [StatsD](https://docs.micrometer.io/micrometer/reference/implementations/statsD.html)
- [CloudWatch](https://docs.micrometer.io/micrometer/reference/implementations/cloudwatch.html)
- [Azure Monitor](https://docs.micrometer.io/micrometer/reference/implementations/azure-monitor.html)
- [Influx](https://docs.micrometer.io/micrometer/reference/implementations/influx.html)

### 启用指标采集

要将指标采集到某个特定的监控系统，请参考 [Micrometer 文档](https://docs.micrometer.io/micrometer/reference/implementations.html) 完成相应实现。此外，你还需要在 Conductor 的 [`application.properties` 文件](https://github.com/conductor-oss/conductor/blob/6147d61d1babf47f5a0a328d114f1eb5d3d5ecb1/server/src/main/resources/application.properties#L163) 中启用对应的监控系统。
