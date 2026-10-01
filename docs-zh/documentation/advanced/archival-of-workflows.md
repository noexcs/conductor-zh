---
description: "工作流归档 —— 自动归档已完成或已终止的 Conductor 工作流，以释放数据库存储。"
---
# 工作流归档

Conductor 支持在工作流终止或完成时对其进行归档。启用此功能后，工作流将从配置的数据库中删除，但关联数据会保留在 Elasticsearch 中，因此仍然可以搜索。 

要启用此功能，将 `conductor.workflow-status-listener.type` 属性设置为 `archive`。

还有一些附加属性可用于控制归档行为。

| 属性                                                                | 默认值 | 说明                                                                                                |
| ------------------------------------------------------------------- | ------ | --------------------------------------------------------------------------------------------------- |
| conductor.workflow-status-listener.archival.ttlDuration             | 0s     | 工作流归档模块的生存时间（以秒为单位）。目前仅 RedisExecutionDAO 支持此属性                          |
| conductor.workflow-status-listener.archival.delayQueueWorkerThreadCount | 5  | 用于处理工作流归档中延迟队列的线程数                                                                  |
| conductor.workflow-status-listener.archival.delaySeconds            | 60     | 延迟工作流归档的时间                                                                                  |
