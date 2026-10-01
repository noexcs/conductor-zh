---
description: "搜索工作流 — 从 CLI、API 或 UI 按名称、状态、关联 ID 或时间范围查找 Conductor 工作流执行。"
---
# 搜索执行

当你知道工作流名称、状态、关联 ID 或时间范围等属性，但不知道工作流 ID 时，使用搜索。

## 使用 CLI 搜索

```bash
conductor workflow search -w order_processing -s FAILED -c 20
conductor workflow search -s COMPLETED \
  --start-time-after "2026-07-01" --start-time-before "2026-07-31"
```

| CLI 选项 | 过滤或控制 | 示例 |
|---|---|---|
| `-w`, `--workflow` | 工作流名称 | `--workflow order_processing` |
| `-s`, `--status` | 执行状态 | `--status FAILED` |
| `-c`, `--count` | 返回的执行数量（最多 1000） | `--count 20` |
| `--start-time-after` | 在指定时间戳之后开始的执行 | `--start-time-after "2026-07-01"` |
| `--start-time-before` | 在指定时间戳之前开始的执行 | `--start-time-before "2026-07-31"` |
| `--json` | 以 JSON 输出代替表格视图 | `--json` |
| `--csv` | 以 CSV 输出代替表格视图 | `--csv` |

结果应包含 `workflowId`、名称、状态和开始时间。在执行恢复操作之前，先用返回的 ID 运行 `conductor workflow get-execution <workflow-id> -c`。

对于超出 CLI 标志范围的结构化/自由文本或基于任务的搜索，使用 `GET /api/workflow/search` 或 `GET /api/workflow/search-by-tasks`；查询语法和分页契约以[工作流 API](../../../documentation/api/workflow.md#search-workflows)为准。

### REST 查询参数

`GET /api/workflow/search` 接受以下查询参数：

| 参数 | 含义 | 默认值 |
|---|---|---|
| `start` | 页偏移 | `0` |
| `size` | 结果数量 | `100` |
| `sort` | 排序顺序，形式为 `<field>:ASC` 或 `<field>:DESC` | 无 |
| `freeText` | 全文搜索查询 | `*` |
| `query` | 类 SQL 过滤表达式 | 无 |
| `classifier` | 按分类器过滤或分组代理工作流执行 | 无 |
| `topLevelOnly` | 将结果限定为顶层工作流执行 | `false` |

## 使用 UI 搜索

在 Conductor UI 中转到**执行 > 工作流**。填写一个或多个过滤器并选择**搜索**。结果可按列排序，**以代码显示**会展示当前过滤条件等效的 `GET /api/workflow/search` 调用。

### 过滤器

| 过滤器 | 说明 |
|---|---|
| 工作流名称 | 一个或多个工作流定义名称。 |
| 工作流 ID | 特定的工作流执行 ID。 |
| 关联 ID | 一个或多个关联 ID。每个值后按 Enter。 |
| 幂等键 | 一个或多个幂等键。每个值后按 Enter。 |
| 状态 | `RUNNING`、`COMPLETED`、`FAILED`、`TIMED_OUT`、`TERMINATED`、`PAUSED` 中的一个或多个。 |
| 开始 / 结束 | 仅选择在所选时间范围内开始的执行。 |
| 自由文本搜索 | 针对已索引工作流数据（如输入和输出值）的全文查询。需要在服务器上启用索引。 |

### SQL 格式

打开 **SQL 格式**，用查询框替换过滤表单，该查询框接受与搜索 API 的 `query` 参数相同的类 SQL 表达式，例如 `workflowType = 'order_processing' AND status = 'FAILED'`。参见[查询语法](../../../documentation/api/workflow.md#query-syntax)。

### 按任务搜索

开源 UI 仅搜索工作流执行。要按工作流包含的任务查找工作流，或直接搜索任务执行，请使用 API：

* `GET /api/workflow/search-by-tasks` — 按任务属性过滤的工作流。参见[按任务搜索](../../../documentation/api/workflow.md#search-by-tasks)。
* `GET /api/tasks/search` — 任务执行。参见[搜索任务](../../../documentation/api/task.md#search-tasks)。

## 限制与下一步

自由文本和任务搜索取决于配置的索引后端及其索引延迟。搜索结果用于识别候选项；在重试、重启或终止执行之前，务必先检查该执行。继续阅读[查看执行](viewing-workflow-executions.md)或[调试与恢复](debugging-workflows.md)。
