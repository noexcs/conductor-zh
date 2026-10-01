---
description: "No Op 任务 — Conductor 工作流中的直通任务，适用于路由、占位步骤和工作流测试。"
---
# No Op 任务
```json
"type" : "NOOP"
```

No Op 任务（NOOP）是一个空操作任务。在 Switch 任务中，对于不需要执行任何动作的分支情况，可以使用它。

## JSON 配置

以下是 No Op 任务的任务配置。

```json
{
	"name": "noop",
    "taskReferenceName": "noop_ref",
	"inputParameters": {},
	"type": "NOOP"
}
```
