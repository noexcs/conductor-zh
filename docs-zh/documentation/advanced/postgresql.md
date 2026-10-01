---
description: "PostgreSQL 后端 —— 配置 Conductor 使用 PostgreSQL 进行工作流持久化、队列、索引和锁。"
---
# PostgreSQL

默认情况下，conductor 使用内存中的 Redis mock 运行。不过，你可以让 Conductor 针对 PostgreSQL 运行，它提供工作流管理、队列、索引和锁。
有一些配置选项，让你可以根据自己的需要使用更多或更少 PostgreSQL 的功能。
它的好处是基础设施所需的活动部件更少，但在处理大量工作流时的扩展性稍逊。
你应该针对自己的具体工作负载，对使用 Postgres 的 Conductor 进行基准测试，以确保无误。


## 配置

要启用使用 PostgreSQL 管理工作流元数据的基本用法，请设置以下属性：

```properties
conductor.db.type=postgres
spring.datasource.url=jdbc:postgresql://postgres:5432/conductor
spring.datasource.username=conductor
spring.datasource.password=password
# optional
conductor.postgres.schema=public
```

如果还想使用 PostgreSQL 作为队列，你可以设置：

```properties
conductor.queue.type=postgres
```

你还可以使用 PostgreSQL 为工作流建立索引，配置如下：

```properties
conductor.indexing.enabled=true
conductor.indexing.type=postgres
conductor.elasticsearch.version=0
```

要使用 PostgreSQL 实现锁，请设置以下配置：
```properties
conductor.app.workflowExecutionLockEnabled=true
conductor.workflow-execution-lock.type=postgres
```

## 性能优化

### 轮询数据缓存

默认情况下，Conductor 会将任务的最新轮询写入数据库，以便用于确定哪些任务和 domain 处于活动状态。这会产生大量数据库流量。
为避免其中一部分流量，你可以为 PollDataDAO 配置一个写缓冲区，使其每 x 毫秒才刷新一次。如果将此值保持在 5s 左右，则应该不会对行为产生影响。Conductor 使用 10s 的默认时长来判定某个 domain 的队列是否处于活动状态（也可使用 `conductor.app.activeWorkerLastPollTimeout` 配置），因此这能确保数据有充足的时间写入数据库，并被其他实例共享：

```properties
# Flush the data every 5 seconds
conductor.postgres.pollDataFlushInterval=5000
```

你还可以配置一个时长，超过该时长后缓存的轮询数据将被视为过期。这意味着 PollDataDAO 会尝试使用缓存数据，但如果数据比配置的周期更旧，它会到数据库中核对。这样设置没有坏处，因为如果该 Conductor 节点已经能够确认队列处于活动状态，则无需访问数据库；如果缓存中的记录已过期，我们仍然会到数据库中核对。

```properties
# Data older than 5 seconds is considered stale
conductor.postgres.pollDataCacheValidityPeriod=5000
```

### 状态变更时的工作流与任务索引

如果有一个包含许多任务的工作流，Conductor 会在每个任务完成时都对该工作流建立索引，这会给数据库带来额外的很多负载。通过设置此参数，你可以配置 Conductor 仅在其状态变更时对工作流建立索引：

```properties
conductor.postgres.onlyIndexOnStatusChange=true
```

### 控制哪些内容被索引

默认情况下，Conductor 会同时为工作流和任务建立索引，以便通过 UI 搜索。如果你发现从不搜索任务、只搜索工作流，可以使用以下选项禁用任务索引：

```properties
conductor.app.taskIndexingEnabled=false
```

### 实验性的基于 LISTEN/NOTIFY 的队列

默认情况下，Conductor 会为每个任务每秒查询数据库中的队列 10 次，这可能会产生大量流量。
启用此选项后，Conductor 会利用 [LISTEN](https://www.postgresql.org/docs/current/sql-listen.html)/[NOTIFY](https://www.postgresql.org/docs/current/sql-notify.html)，通过触发器将队列状态的元数据分发给所有 Conductor 服务器。由于一条包含队列状态的消息会发送给所有订阅者，因此这大幅降低了数据库的负载。
按如下方式启用：

```properties
conductor.postgres.experimentalQueueNotify=true
```

你还可以使用以下属性，配置 Conductor 在认为通知过期之前等待多久：

```properties
# Data older than 5 seconds is considered stale
conductor.postgres.experimentalQueueNotifyStalePeriod=5000
```
