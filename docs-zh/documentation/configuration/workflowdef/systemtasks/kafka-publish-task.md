---
description: "Kafka Publish 任务 — 从 Conductor 工作流向 Kafka 主题发送消息，支持可配置的序列化。"
---
# Kafka Publish 任务
```json
"type" : "KAFKA_PUBLISH"
```

Kafka Publish 任务（`KAFKA_PUBLISH`）用于通过 Kafka 向另一个微服务推送消息。

## 任务参数
该任务期望在任务的 `inputParameters` 中包含一个名为 `kafka_request` 的字段。

在 Kafka Publish 任务配置中，在 `inputParameters` 内使用这些参数。

| 参数          | 类型                | 说明                                       | 必填 / 可选  |
| ------------------ | ------------------- | ------------------------------------------------- | -------------------- |
| kafka_request | KafkaRequest | 包含 bootstrap server、消息等内容的 JSON 对象。 | Required. |
| kafka_request.bootStrapServers | String        | 用于连接 Kafka 集群的 bootstrap server。             | Required.     |
| kafka_request.topic            | String        | 要发布消息的主题。                                  | Required.     |
| kafka_request.value            | Any        | 要发布的消息。                                         | Required.     |
| kafka_request.key              | String        | Kafka 消息的 key。具有相同 key 的消息将被发送到同一主题分区。             | Optional.     |
| kafka_request.keySerializer    | String (enum) | 用于序列化消息 key 的序列化器。默认为 `StringSerializer`。支持的取值：<ul><li>`org.apache.kafka.common.serialization.IntegerSerializer`</li> <li>`org.apache.kafka.common.serialization.LongSerializer`</li> <li>`org.apache.kafka.common.serialization.StringSerializer`</li></ul> | Optional.     |
| kafka_request.headers          | Map[String, Any]  | 随 Kafka 消息一起发送的其他请求头。                     | Optional.     |
| kafka_request.requestTimeoutMs | Integer     | 等待响应时的请求超时时间（毫秒）。          | Optional.   |
| kafka_request.maxBlockMs       | Integer     | 发布到 Kafka 时的最大阻塞时间。                  | Optional.   |

## JSON 配置

以下是 Kafka Publish 任务的任务配置。

```json
{
  "name": "kafka",
  "taskReferenceName": "kafka_ref",
  "inputParameters": {
    "kafka_request": {
      "topic": "userTopic",
      "value": "Message to publish",
      "bootStrapServers": "localhost:9092",
      "headers": {
        "x-Auth":"Auth-key"    
      },
      "key": "123",
      "keySerializer": "org.apache.kafka.common.serialization.IntegerSerializer"
    }
  },
  "type": "KAFKA_PUBLISH"
}
```

## 输出

如果消息成功发布到 Kafka 队列，任务会转至 COMPLETED 状态；如果消息无法发布，则标记为 FAILED。
