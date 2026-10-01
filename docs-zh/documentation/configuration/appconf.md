---
description: "Conductor 应用服务器配置——调优持久化执行、工作流引擎可扩展性、系统任务工作者以及生产部署设置。"
---

# 应用配置

Conductor 应用服务器提供了丰富的自定义选项，用于针对特定环境优化其运行。

这些配置参数允许对服务器行为、性能与集成能力的各个方面进行精细调优。
所有这些参数都归属于 `conductor.app` 命名空间。

### 配置

| 字段                                       | 类型     | 描述                                                                                                                                  | 说明                                                  |
|:--------------------------------------------|:---------|:---------------------------------------------------------------------------------------------------------------------------------------|:-------------------------------------------------------|
| stack                                       | String   | 应用所运行 stack 的名称。例如 `devint`、`testintg`、`staging`、`prod` 等。                                                            | 默认为 "test"                                          |
| appId                                       | String   | 应用注册时所使用的 ID。例如 `conductor`、`myApp`                                                                                      | 默认为 "conductor"                                     |
| executorServiceMaxThreadCount               | int      | 分配给执行器服务线程池的最大线程数。例如 `50`                                                                                          | 默认为 50                                              |
| workflowOffsetTimeout                       | Duration | 工作流被推送到 decider 队列时要设置的超时时长。示例：`30s` 或 `1m`                                                                     | 默认为 30 秒                                           |
| maxPostponeDurationSeconds                  | Duration | 带有运行中任务的工作流被推送到 decider 队列时要设置的最大超时时长。示例：`30m` 或 `1h`                                                 | 默认为 3600 秒                                         |
| sweeperThreadCount                          | int      | 用于对活跃工作流进行后台清扫的线程数。例如，如果有 4 个处理器，则为 `8`（2x4）                                                         | 默认为可用处理器数量的 2 倍                            |
| sweeperWorkflowPollTimeout                  | Duration | 轮询待清扫工作流的超时时间。示例：`2000ms` 或 `2s`                                                                                     | 默认为 2000 毫秒                                       |
| eventProcessorThreadCount                   | int      | 用于配置事件处理器线程池的线程数。示例：`4`                                                                                             | 默认为 2                                               |
| eventMessageIndexingEnabled                 | boolean  | 是否启用对事件负载中消息的索引。示例：`true` 或 `false`                                                                                | 默认为 true                                            |
| eventExecutionIndexingEnabled               | boolean  | 是否启用对事件执行结果的索引。示例：`true` 或 `false`                                                                                  | 默认为 true                                            |
| workflowExecutionLockEnabled                | boolean  | 是否启用工作流执行锁。示例：`true` 或 `false`                                                                                          | 默认为 false                                           |
| lockLeaseTime                               | Duration | 锁的租约时长。示例：`60000ms` 或 `1m`                                                                                                  | 默认为 60000 毫秒                                      |
| lockTimeToTry                               | Duration | 线程尝试获取锁时阻塞的时长。示例：`500ms` 或 `1s`                                                                                      | 默认为 500 毫秒                                        |
| activeWorkerLastPollTimeout                 | Duration | 判定工作者是否正在积极轮询任务的时间。示例：`10s`                                                                                      | 默认为 10 秒                                           |
| taskExecutionPostponeDuration               | Duration | 任务执行因限流或并发执行受限而被推迟的时长。示例：`60s`                                                                                 | 默认为 60 秒                                           |
| taskIndexingEnabled                         | boolean  | 是否启用任务索引。示例：`true` 或 `false`                                                                                              | 默认为 true                                            |
| taskExecLogIndexingEnabled                  | boolean  | 是否启用任务执行日志索引。示例：`true` 或 `false`                                                                                      | 默认为 true                                            |
| asyncIndexingEnabled                        | boolean  | 是否启用向 Elasticsearch 的异步索引。示例：`true` 或 `false`                                                                           | 默认为 false                                           |
| systemTaskWorkerThreadCount                 | int      | 系统任务工作者线程池中的线程数。例如，如果有 4 个处理器，则为 `8`（2x4）                                                                | 默认为可用处理器数量的 2 倍                            |
| systemTaskMaxPollCount                      | int      | 系统任务工作者线程池内参与轮询的最大线程数。示例：`8`                                                                                   | 默认等于 systemTaskWorkerThreadCount                   |
| systemTaskWorkerCallbackDuration            | Duration | 系统任务工作者检查系统任务是否完成的间隔。示例：`30s`                                                                                   | 默认为 30 秒                                           |
| systemTaskWorkerPollInterval                | Duration | 系统任务工作者轮询系统任务队列的间隔。示例：`50ms`                                                                                      | 默认为 50 毫秒                                         |
| systemTaskWorkerExecutionNamespace          | String   | 系统任务工作者用于提供实例级隔离的命名空间。示例：`namespace1`、`namespace2`                                                            | 默认为空字符串                                         |
| isolatedSystemTaskWorkerThreadCount         | int      | 每个隔离组中系统任务工作者线程池使用的线程数。示例：`4`                                                                                 | 默认为 1                                               |
| taskWorkerConfigs                           | Map      | 按任务类型划分的 worker 覆盖配置，以任务类型作为键。`<TYPE>.threadCount` > 0 时，该类型将拥有大小为此值的专属线程池，该值同时也是其并发上限——同一时刻最多并发运行 `threadCount` 个该类型的任务。没有配置条目的类型共享公共线程池。示例：`taskWorkerConfigs.HTTP.threadCount=20` | 默认为空（所有类型共享公共线程池）                     |
| asyncUpdateShortRunningWorkflowDuration     | Duration | 启用向 Elasticsearch 的异步索引时，被视为短运行期的工作流执行时长标准。示例：`30s`                                                      | 默认为 30 秒                                           |
| asyncUpdateDelay                            | Duration | 启用异步索引时，短运行期工作流在 Elasticsearch 中更新的延迟。示例：`60s`                                                                 | 默认为 60 秒                                           |
| ownerEmailMandatory                         | boolean  | 是否在工作流定义和任务定义中将 owner 邮箱字段作为必填项进行校验。示例：`true` 或 `false`                                                | 默认为 true                                            |
| eventQueueSchedulerPollThreadCount          | int      | 调度器中用于从多个事件队列轮询事件的线程数。例如，如果有 4 个处理器，则为 `8`（2x4）                                                    | 默认为等于可用处理器数量                               |
| eventQueuePollInterval                      | Duration | 轮询默认事件队列的时间间隔。示例：`100ms`                                                                                               | 默认为 100 毫秒                                        |
| eventQueuePollCount                         | int      | 单次操作从默认事件队列中轮取的消息数。示例：`10`                                                                                        | 默认为 10                                              |
| eventQueueLongPollTimeout                   | Duration | 对默认事件队列执行轮询操作的超时时间。示例：`1000ms`                                                                                    | 默认为 1000 毫秒                                       |
| workflowInputPayloadSizeThreshold           | DataSize | 工作流输入负载大小的阈值，超过该阈值的负载将被存储到 ExternalPayloadStorage。示例：`5120KB`                                            | 默认为 5120 千字节                                     |
| maxWorkflowInputPayloadSizeThreshold        | DataSize | 工作流输入负载大小的最大阈值，超过该阈值的输入将被拒绝，并且工作流被标记为 FAILED。示例：`10240KB`                                      | 默认为 10240 千字节                                    |
| workflowOutputPayloadSizeThreshold          | DataSize | 工作流输出负载大小的阈值，超过该阈值的负载将被存储到 ExternalPayloadStorage。示例：`5120KB`                                            | 默认为 5120 千字节                                     |
| maxWorkflowOutputPayloadSizeThreshold       | DataSize | 工作流输出负载大小的最大阈值，超过该阈值的输出将被拒绝，并且工作流被标记为 FAILED。示例：`10240KB`                                      | 默认为 10240 千字节                                    |
| taskInputPayloadSizeThreshold               | DataSize | 任务输入负载大小的阈值，超过该阈值的负载将被存储到 ExternalPayloadStorage。示例：`3072KB`                                              | 默认为 3072 千字节                                     |
| maxTaskInputPayloadSizeThreshold            | DataSize | 任务输入负载大小的最大阈值，超过该阈值的任务输入将被拒绝，并且任务被标记为 FAILED_WITH_TERMINAL_ERROR。示例：`10240KB`                  | 默认为 10240 千字节                                    |
| taskOutputPayloadSizeThreshold              | DataSize | 任务输出负载大小的阈值，超过该阈值的负载将被存储到 ExternalPayloadStorage。示例：`3072KB`                                              | 默认为 3072 千字节                                     |
| maxTaskOutputPayloadSizeThreshold           | DataSize | 任务输出负载大小的最大阈值，超过该阈值的任务输出将被拒绝，并且任务被标记为 FAILED_WITH_TERMINAL_ERROR。示例：`10240KB`                  | 默认为 10240 千字节                                    |
| maxWorkflowVariablesPayloadSizeThreshold    | DataSize | 工作流变量负载大小的最大阈值，超过该阈值的任务变更将被拒绝，并且任务被标记为 FAILED_WITH_TERMINAL_ERROR。示例：`256KB`                 | 默认为 256 千字节                                      |
| taskExecLogSizeLimit                        | int      | 任务执行日志的最大大小。示例：`10000`                                                                                                   | 默认为 10                                              |

### 使用示例

在配置文件中，按需添加所需配置

```properties
# Conductor App Configuration

# Name of the stack within which the app is running. e.g. devint, testintg, staging, prod etc.
conductor.app.stack=test

# The ID with which the app has been registered. e.g. conductor, myApp
conductor.app.appId=conductor

# The maximum number of threads to be allocated to the executor service threadpool. e.g. 50
conductor.app.executorServiceMaxThreadCount=50

# The timeout duration to set when a workflow is pushed to the decider queue. Example: 30s or 1m
conductor.app.workflowOffsetTimeout=30s

# The number of threads to use for background sweeping on active workflows. Example: 8 if there are 4 processors (2x4)
conductor.app.sweeperThreadCount=8

# The timeout for polling workflows to be swept. Example: 2000ms or 2s
conductor.app.sweeperWorkflowPollTimeout=2000ms

# The number of threads to configure the threadpool in the event processor. Example: 4
conductor.app.eventProcessorThreadCount=4

# Whether to enable indexing of messages within event payloads. Example: true or false
conductor.app.eventMessageIndexingEnabled=true

# Whether to enable indexing of event execution results. Example: true or false
conductor.app.eventExecutionIndexingEnabled=true

# Whether to enable the workflow execution lock. Example: true or false
conductor.app.workflowExecutionLockEnabled=false

# The time for which the lock is leased. Example: 60000ms or 1m
conductor.app.lockLeaseTime=60000ms

# The time for which the thread will block in an attempt to acquire the lock. Example: 500ms or 1s
conductor.app.lockTimeToTry=500ms

# The time to consider if a worker is actively polling for a task. Example: 10s
conductor.app.activeWorkerLastPollTimeout=10s

# The time for which a task execution will be postponed if rate-limited or concurrent execution limited. Example: 60s
conductor.app.taskExecutionPostponeDuration=60s

# Whether to enable indexing of tasks. Example: true or false
conductor.app.taskIndexingEnabled=true

# Whether to enable indexing of task execution logs. Example: true or false
conductor.app.taskExecLogIndexingEnabled=true

# Whether to enable asynchronous indexing to Elasticsearch. Example: true or false
conductor.app.asyncIndexingEnabled=false

# The number of threads in the threadpool for system task workers. Example: 8 if there are 4 processors (2x4)
conductor.app.systemTaskWorkerThreadCount=8

# The maximum number of threads to be polled within the threadpool for system task workers. Example: 8
conductor.app.systemTaskMaxPollCount=8

# The interval after which a system task will be checked by the system task worker for completion. Example: 30s
conductor.app.systemTaskWorkerCallbackDuration=30s

# The interval at which system task queues will be polled by system task workers. Example: 50ms
conductor.app.systemTaskWorkerPollInterval=50ms

# The namespace for the system task workers to provide instance-level isolation. Example: namespace1, namespace2
conductor.app.systemTaskWorkerExecutionNamespace=

# The number of threads to be used within the threadpool for system task workers in each isolation group. Example: 4
conductor.app.isolatedSystemTaskWorkerThreadCount=4

# Per-task-type system task worker overrides. threadCount > 0 gives the task type its own
# dedicated thread pool of that size, which is also its in-flight cap
#conductor.app.taskWorkerConfigs.HTTP.threadCount=20
#conductor.app.taskWorkerConfigs.LLM_TEXT_COMPLETE.threadCount=4

# The duration of workflow execution qualifying as short-running when async indexing to Elasticsearch is enabled. Example: 30s
conductor.app.asyncUpdateShortRunningWorkflowDuration=30s

# The delay with which short-running workflows will be updated in Elasticsearch when async indexing is enabled. Example: 60s
conductor.app.asyncUpdateDelay=60s

# Whether to validate the owner email field as mandatory within workflow and task definitions. Example: true or false
conductor.app.ownerEmailMandatory=true

# The number of threads used in the Scheduler for polling events from multiple event queues. Example: 8 if there are 4 processors (2x4)
conductor.app.eventQueueSchedulerPollThreadCount=8

# The time interval at which the default event queues will be polled. Example: 100ms
conductor.app.eventQueuePollInterval=100ms

# The number of messages to be polled from a default event queue in a single operation. Example: 10
conductor.app.eventQueuePollCount=10

# The timeout for the poll operation on the default event queue. Example: 1000ms
conductor.app.eventQueueLongPollTimeout=1000ms

# The threshold of the workflow input payload size beyond which the payload will be stored in ExternalPayloadStorage. Example: 5120KB
conductor.app.workflowInputPayloadSizeThreshold=5120KB

# The maximum threshold of the workflow input payload size beyond which input will be rejected and the workflow marked as FAILED. Example: 10240KB
conductor.app.maxWorkflowInputPayloadSizeThreshold=10240KB

# The threshold of the workflow output payload size beyond which the payload will be stored in ExternalPayloadStorage. Example: 5120KB
conductor.app.workflowOutputPayloadSizeThreshold=5120KB

# The maximum threshold of the workflow output payload size beyond which output will be rejected and the workflow marked as FAILED. Example: 10240KB
conductor.app.maxWorkflowOutputPayloadSizeThreshold=10240KB

# The threshold of the task input payload size beyond which the payload will be stored in ExternalPayloadStorage. Example: 3072KB
conductor.app.taskInputPayloadSizeThreshold=3072KB

# The maximum threshold of the task input payload size beyond which the task input will be rejected and the task marked as FAILED_WITH_TERMINAL_ERROR. Example: 10240KB
conductor.app.maxTaskInputPayloadSizeThreshold=10240KB

# The threshold of the task output payload size beyond which the payload will be stored in ExternalPayloadStorage. Example: 3072KB
conductor.app.taskOutputPayloadSizeThreshold=3072KB

# The maximum threshold of the task output payload size beyond which the task output will be rejected and the task marked as FAILED_WITH_TERMINAL_ERROR. Example: 10240KB
conductor.app.maxTaskOutputPayloadSizeThreshold=10240KB

# The maximum threshold of the workflow variables payload size beyond which the task changes will be rejected and the task marked as FAILED_WITH_TERMINAL_ERROR. Example: 256KB
conductor.app.maxWorkflowVariablesPayloadSizeThreshold=256KB

# The maximum size of task execution logs. Example: 10000
conductor.app.taskExecLogSizeLimit=10000
```
