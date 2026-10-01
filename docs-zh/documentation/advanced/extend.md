---
description: "使用自定义持久化后端、队列实现和工作流状态监听器，扩展这个开源工作流编排引擎 Conductor。"
---

# 扩展 Conductor

## 后端
Conductor 提供可插拔的后端。支持的实现包括 Redis、PostgreSQL、MySQL、Cassandra 和 SQLite。

每个后端需要实现 4 个接口：

```java
//Store for workflow and task definitions
com.netflix.conductor.dao.MetadataDAO
```

```java
//Store for workflow executions
com.netflix.conductor.dao.ExecutionDAO
```

```java
//Index for workflow executions
com.netflix.conductor.dao.IndexDAO
```

```java
//Queue provider for tasks
com.netflix.conductor.dao.QueueDAO
```

可以为其中每一项混合搭配不同的实现。  
例如，队列使用 SQS，其他使用关系型存储。


## 系统任务
创建系统任务请遵循以下步骤：

* 继承 ```com.netflix.conductor.core.execution.tasks.WorkflowSystemTask```
* 在启动时实例化新类（饿汉单例）
* 实现 ```TaskMapper``` [接口](https://github.com/conductor-oss/conductor/blob/main/core/src/main/java/com/netflix/conductor/core/execution/mapper/TaskMapper.java)

## 工作流状态监听器 { #workflow-status-listener }
要在工作流完成/终止时提供通知机制：

* 实现 ```WorkflowStatusListener``` [接口](https://github.com/conductor-oss/conductor/blob/main/core/src/main/java/com/netflix/conductor/core/listener/WorkflowStatusListener.java)
* 可以配置为在工作流到达终止状态时插入自定义通知/事件机制。

## 事件处理
提供 [EventQueueProvider](https://github.com/conductor-oss/conductor/blob/main/core/src/main/java/com/netflix/conductor/core/events/EventQueueProvider.java) 的实现。

例如 SQS 队列提供者： 
[SQLinkEventQueueProvider.java ](https://github.com/conductor-oss/conductor/blob/main/awssqs-event-queue/src/main/java/com/netflix/conductor/sqs/config/SQSEventQueueProvider.java)