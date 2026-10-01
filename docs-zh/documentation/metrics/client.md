---
description: "客户端指标 — 使用内置的任务轮询和执行指标来监控 Conductor Java 客户端的性能。"
---
# 客户端指标

使用 Java 客户端时，会发布以下指标：

| 名称        | 用途           | 标签  |
| ------------- |:-------------| -----|
| task_execution_queue_full | 记录执行队列已饱和的计数器 | taskType|
| task_poll_error | 轮询任务队列时发生的客户端错误 | taskType, includeRetries, status |
| task_paused | 工作者处于暂停状态时，任务被轮询次数的计数器 | taskType |
| task_execute_error | 执行错误 | taskType|
| task_ack_failed | 任务 ack 失败 | taskType |
| task_ack_error | 任务 ack 遇到异常 | taskType |
| task_update_error | 任务状态无法更新回服务器  | taskType |
| task_poll_counter | 每次执行轮询时递增  | taskType |
| task_poll_time | 轮询一批任务所花费的时间 | taskType |
| task_execute_time | 执行一个任务所花费的时间  | taskType |
| task_result_size | 记录任务的输出 payload 大小 | taskType |
| workflow_input_size | 记录工作流的输入 payload 大小 | workflowType, workflowVersion |
| external_payload_used | 每次使用外部 payload 存储时递增 | name, operation, payloadType | 

客户端指标是对服务器端采集指标的补充，有助于识别网络问题以及客户端侧的问题。

[1]: https://github.com/Netflix/spectator
