---
description: "EXCLUSIVE_JOIN 操作符 — 当第一个选定分支完成时继续。"
---

# Exclusive Join

```json
"type": "EXCLUSIVE_JOIN"
```

`EXCLUSIVE_JOIN` 等待 `joinOn` 中的第一个任务完成，而不是像 `JOIN` 那样等待所有分支。它适用于竞争（race）或回退（fallback）模式。

## 配置

| 字段 | 必填 | 描述 |
|---|---:|---|
| `joinOn` | 是 | 可能满足汇聚条件的任务引用名列表。 |
| `defaultExclusiveJoinTask` | 否 | 当列出的任务都没有被选中时使用的回退任务引用列表。 |

```json
{
  "name": "first_response",
  "taskReferenceName": "first_response",
  "type": "EXCLUSIVE_JOIN",
  "joinOn": ["primary_response", "fallback_response"],
  "defaultExclusiveJoinTask": ["fallback_response"]
}
```

映射器会将 `joinOn` 以及（如果存在时）`defaultExclusiveJoinTask` 传递给运行时汇聚任务。当所有分支都必须完成时，请使用普通的 [Join](join-task.md)。
