---
description: "Conductor 手册（cookbook）— 等待与定时器模式示例集，涵盖固定延迟、定时执行、外部信号和人在回路（HITL）审批。"
---

# 等待与定时器模式

### 等待固定延迟

在工作流步骤之间引入延迟——适用于限流、冷却期或重试退避。

```json
{
  "name": "delayed_notification",
  "version": 1,
  "schemaVersion": 2,
  "tasks": [
    {
      "name": "process_event",
      "taskReferenceName": "process",
      "type": "SIMPLE"
    },
    {
      "name": "wait_before_retry",
      "taskReferenceName": "cooldown",
      "type": "WAIT",
      "inputParameters": {
        "duration": "5 minutes"
      }
    },
    {
      "name": "send_notification",
      "taskReferenceName": "notify",
      "type": "HTTP",
      "inputParameters": {
        "uri": "https://api.example.com/notify",
        "method": "POST",
        "body": {"eventId": "${process.output.eventId}"}
      }
    }
  ]
}
```

`duration` 字段支持人类可读的格式：`30 seconds`、`5 minutes`、`2 hours`、`1 days`，或 `30s`、`5m`、`2h`、`1d` 之类的简短形式。也可以组合使用：`2 hours 30 minutes`。

---

### 等待到特定时间

把工作流的继续执行安排在特定日期/时间——适用于计划内发布、SLA 截止时间或工作时间处理。

```json
{
  "name": "scheduled_report",
  "version": 1,
  "schemaVersion": 2,
  "inputParameters": ["reportDate"],
  "tasks": [
    {
      "name": "prepare_report",
      "taskReferenceName": "prepare",
      "type": "SIMPLE"
    },
    {
      "name": "wait_until_publish_time",
      "taskReferenceName": "schedule_wait",
      "type": "WAIT",
      "inputParameters": {
        "until": "${workflow.input.reportDate}"
      }
    },
    {
      "name": "publish_report",
      "taskReferenceName": "publish",
      "type": "HTTP",
      "inputParameters": {
        "uri": "https://api.example.com/reports/publish",
        "method": "POST",
        "body": {"reportId": "${prepare.output.reportId}"}
      }
    }
  ]
}
```

`until` 字段支持的格式：`yyyy-MM-dd HH:mm z`（例如 `2025-06-15 09:00 GMT+00:00`）、`yyyy-MM-dd HH:mm` 或 `yyyy-MM-dd`。

**注册并运行：**

```shell
curl -X POST 'http://localhost:8080/api/metadata/workflow' \
  -H 'Content-Type: application/json' \
  -d @scheduled_report.json

curl -X POST 'http://localhost:8080/api/workflow/scheduled_report' \
  -H 'Content-Type: application/json' \
  -d '{"reportDate": "2025-06-15 09:00 GMT+00:00"}'
```

---

### 等待外部信号

暂停工作流，直到外部系统（或人）通过 API 完成任务——适用于审批、人工 QA 或第三方回调。

```json
{
  "name": "order_with_manual_approval",
  "version": 1,
  "schemaVersion": 2,
  "inputParameters": ["orderId", "amount"],
  "tasks": [
    {
      "name": "validate_order",
      "taskReferenceName": "validate",
      "type": "HTTP",
      "inputParameters": {
        "uri": "https://api.example.com/orders/${workflow.input.orderId}/validate",
        "method": "GET"
      }
    },
    {
      "name": "wait_for_approval",
      "taskReferenceName": "approval",
      "type": "WAIT"
    },
    {
      "name": "fulfill_order",
      "taskReferenceName": "fulfill",
      "type": "HTTP",
      "inputParameters": {
        "uri": "https://api.example.com/orders/${workflow.input.orderId}/fulfill",
        "method": "POST",
        "body": {
          "approvedBy": "${approval.output.approvedBy}"
        }
      }
    }
  ]
}
```

在外部完成 WAIT 任务（例如从 UI 或 webhook）：

```shell
# Complete the currently blocked wait task and return the updated workflow
curl -X POST 'http://localhost:8080/api/tasks/{workflowId}/COMPLETED/signal/sync' \
  -H 'Content-Type: application/json' \
  -d '{"approvedBy": "manager@example.com"}'
```

你在向当前被阻塞的 `WAIT` 任务发送信号时传入的输出数据，可在后续任务中通过 `${approval.output.approvedBy}` 使用。关于异步信号、返回策略和超时行为，参见[向工作流发送信号](sending-signals.md)。
