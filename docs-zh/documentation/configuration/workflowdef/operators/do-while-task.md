---
description: "Do While 任务 — 在 Conductor 工作流中循环执行任务直到满足条件，支持可配置的迭代次数限制。"
---
# Do While
```json
"type" : "DO_WHILE"
```

在给定条件为真的期间，Do While 任务（`DO_WHILE`）会顺序执行一个任务列表。任务序列总是先执行、之后再检查条件，即使是第一次迭代也是如此，这与编程中常规的 _do.. while_ 语句相同。

## 任务参数

在 Do While 任务配置顶层使用以下参数。

| 参数          | 类型                | 描述                                       | 必填 / 可选  |
| ------------------ | ------------------- | ------------------------------------------------- | -------------------- |
| loopCondition | String      | 每次迭代之后评估的条件。这是一个 JavaScript 表达式，使用 Nashorn 引擎进行评估。使用 `items` 进行列表迭代时，此参数是可选的。 | 必填（基于计数器的迭代）。<br/>可选（列表迭代）。 |
| loopOver      | List[Task] | 在条件为真期间将执行的任务配置列表。                                                                                                                                               | 必填。 |
| items         | String      | 一个求值为待迭代列表/数组的工作流表达式（例如 `${workflow.input.myList}`）。指定后，循环会自动遍历每个项，无需 `loopCondition`。循环任务可通过 `${do_while_ref.output.loopItem}` 访问当前项，通过 `${do_while_ref.output.loopIndex}` 访问从零开始的索引。 | 可选。 |

## 输入参数

在 Do While 任务配置的 `inputParameters` 部分使用以下参数。

| 参数     | 类型    | 描述                                                                                                                                                                                                                                                                                        | 必填 / 可选 |
| ------------- | ------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------- |
| keepLastN     | Integer | 要保留在数据库和任务输出中的最近迭代次数。较早的迭代会被自动移除，以防止数据库膨胀。未指定时，保留所有迭代（默认行为）。这对迭代次数众多的长时间运行循环很有用。最小值：1。 | 可选。           |

## JSON 配置

以下是 Do While 任务的任务配置。

**基于计数器的迭代：**
```json
{
  "name": "do_while",
  "taskReferenceName": "do_while_ref",
  "inputParameters": {
    "keepLastN": 10
  },
  "type": "DO_WHILE",
  "loopCondition": "(function () {\n  if ($.do_while_ref['iteration'] < 5) {\n    return true;\n  }\n  return false;\n})();",
  "loopOver": [ // List of tasks to be executed in the loop
    {
        // task configuration
    },
    {
        // task configuration
    }
  ]
}
```

**列表迭代：**
```json
{
  "name": "do_while",
  "taskReferenceName": "do_while_ref",
  "type": "DO_WHILE",
  "items": "${workflow.input.myList}",
  "loopOver": [
    {
      "name": "process_task",
      "taskReferenceName": "process_ref",
      "type": "SIMPLE",
      "inputParameters": {
        "item": "${do_while_ref.output.loopItem}",
        "index": "${do_while_ref.output.loopIndex}"
      }
    }
  ]
}
```

## 输出

Do While 任务将返回以下参数。

| 名称             | 类型         | 描述                                                   |
| ---------------- | ------------ | ------------------------------------------------------------- |
| iteration | Integer          | 迭代次数。<br/><br/> 如果 Do While 任务正在进行中，`iteration` 将显示当前迭代编号。完成时，`iteration` 将显示最终迭代次数。                          |
| loopItem | Any | **（仅限列表迭代）** 本次迭代中来自 `items` 列表的当前项。使用 `items` 参数时可用。 |
| loopIndex | Integer | **（仅限列表迭代）** 当前项的从零开始的索引（0, 1, 2, ...）。使用 `items` 参数时可用。 |

此外，会为每次迭代创建一个映射，以迭代编号作为键（例如 1, 2, 3），其中包含所有 `loopOver` 任务的输出。

### 在 `loopCondition` 中读取状态

循环条件在其 `taskReferenceName` 下接收循环任务的输出，当前循环迭代中的每个任务直接在其自身的 `taskReferenceName` 下。循环体任务的值是其输出数据映射；它们不会被包裹在额外的 `output` 对象中。

```javascript
// `loop` is the DO_WHILE reference; `check` is a loop-body task reference.
if ($.check['done'] == true || $.loop['iteration'] >= 10) { false; } else { true; }
```

在已完成的任务输出中，迭代数据仍然可以在诸如 `loop.output.1.check` 这样的数字键下访问。上述直接绑定仅在评估 `loopCondition` 时适用。

## 执行

执行 Do While 循环时，循环中每个任务的 `taskReferenceName` 都会与 _\_\_i_ 拼接，其中 _i_ 为从 1 开始的迭代编号。如果其中一个循环任务失败，Do While 任务状态将被设为 FAILED，重试时迭代编号将从 1 重新开始。

每个循环任务的输出作为 Do While 任务的一部分存储，以迭代值作为索引，使 `loopCondition` 能够引用特定迭代中任务的输出（例如 `$.LoopTask['iteration']['first_task']`）。


## 迭代清理

对于迭代次数很多（例如 100 次以上）的 Do While 循环，存储所有迭代数据可能会导致数据库膨胀、内存耗尽和性能下降。`keepLastN` 输入参数提供对旧迭代的自动清理。

**工作原理：**

当在 `inputParameters` 中指定 `keepLastN` 时，一旦迭代次数超过 `keepLastN` 值，Conductor 会自动从数据库和任务输出中移除旧的迭代数据。例如，当 `keepLastN: 5` 时：

- 迭代 1-5：保留所有迭代
- 迭代 6：移除迭代 1，保留迭代 2-6
- 迭代 7：移除迭代 2，保留迭代 3-7
- 依此类推……

**重要注意事项：**

- **选择性启用：** 仅当明确设置 `keepLastN` 时才会执行清理。没有此参数时，所有迭代都会被保留（默认行为）。
- **向后兼容性：** 不含 `keepLastN` 的现有工作流继续按原样工作。
- **输出数据：** 任务输出中只有最近 N 次迭代可用。较早的迭代将被永久移除。
- **循环条件：** 如果使用 `keepLastN`，请确保 `loopCondition` 只引用最近几次的迭代，因为较早的迭代数据将不可用。
- **最佳实践：**
  - 对于预计运行 100 次以上迭代的循环，考虑将 `keepLastN` 设置为合理的值（例如 5-10）。
  - 选择一个既能平衡内存使用，又能满足访问历史迭代数据需求的 `keepLastN` 值。
  - 如果 `loopCondition` 需要引用较早的迭代，请确保 `keepLastN` 设置得足够高以保留这些数据。

**带清理的示例：**

```json
{
  "name": "long_running_loop",
  "taskReferenceName": "long_running_loop_ref",
  "inputParameters": {
    "keepLastN": 5
  },
  "type": "DO_WHILE",
  "loopCondition": "if ($.long_running_loop_ref['iteration'] < 1000) { true; } else { false; }",
  "loopOver": [
    {
      "name": "process_item",
      "taskReferenceName": "process_item_ref",
      "type": "SIMPLE"
    }
  ]
}
```

在此示例中，尽管循环运行 1000 次迭代，但在任意时刻数据库中只保留最后 5 次迭代，从而防止数据库膨胀。

## 示例

以下是使用 Do While 任务的一些示例。

### 列表迭代（简化方法）

当有一个需要迭代的项列表时，使用 `items` 参数可以获得更简单的方法，无需手动管理计数器。

```json
{
  "name": "process_items",
  "taskReferenceName": "process_items_ref",
  "type": "DO_WHILE",
  "items": "${workflow.input.itemList}",
  "loopOver": [
    {
      "name": "http",
      "taskReferenceName": "http_ref",
      "inputParameters": {
        "http_request": {
          "uri": "https://api.example.com/process",
          "method": "POST",
          "body": {
            "item": "${process_items_ref.output.loopItem}",
            "index": "${process_items_ref.output.loopIndex}"
          }
        }
      },
      "type": "HTTP"
    }
  ]
}
```

在此示例中：
- 循环自动遍历 `workflow.input.itemList` 中的每个项
- `loopItem` 包含当前项（例如第一次迭代得到 `itemList[0]`）
- `loopIndex` 包含从零开始的索引（0, 1, 2, ...）
- 无需 `loopCondition` —— 所有项处理完毕后循环即停止
- 如果输入列表为空（`[]`），Do While 任务将立即完成，不执行任何循环任务

**与列表迭代结合的可选条件：**

可以将 `items` 与 `loopCondition` 结合使用，以添加提前终止逻辑：

```json
{
  "name": "process_until_error",
  "taskReferenceName": "process_ref",
  "type": "DO_WHILE",
  "items": "${workflow.input.tasks}",
  "loopCondition": "$.http_ref['response']['status'] == 'success'",
  "loopOver": [
    {
      "name": "http",
      "taskReferenceName": "http_ref",
      "type": "HTTP",
      "inputParameters": {
        "http_request": {
          "uri": "${process_ref.output.loopItem.url}",
          "method": "GET"
        }
      }
    }
  ]
}
```

该循环将在所有项处理完毕后停止，或者在 HTTP 响应状态不是 'success' 时停止。

### 使用基本脚本（基于计数器的迭代）

在此示例任务配置中，Do While 任务评估两个条件：


```json
{
    "name": "Loop",
    "taskReferenceName": "LoopTask",
    "type": "DO_WHILE",
    "inputParameters": {
      "value": "${workflow.input.value}"
    },
    "loopCondition": "if ( ($.LoopTask['iteration'] < $.value ) || ( $.first_task['response']['body'] > 10)) { false; } else { true; }",
    "loopOver": [
        {
            "name": "firstTask",
            "taskReferenceName": "first_task",
            "inputParameters": {
                "http_request": {
                    "uri": "http://localhost:8082",
                    "method": "POST"
                }
            },
            "type": "HTTP"
        },{
            "name": "secondTask",
            "taskReferenceName": "second_task",
            "inputParameters": {
                "http_request": {
                    "uri": "http://localhost:8082",
                    "method": "POST"
                }
            },
            "type": "HTTP"
        }
    ],
    "startDelay": 0,
    "optional": false
}
```

假设发生了三次执行（`first_task__1`、`first_task__2`、`first_task__3`、
`second_task__1`、`second_task__2` 和 `second_task__3`），Do While 任务将产生以下输出：

```json
{
    "iteration": 3,
    "1": {
        "first_task": {
            "response": {},
            "headers": {
                "Content-Type": "application/json"
            }
        },
        "second_task": {
            "response": {},
            "headers": {
                "Content-Type": "application/json"
            }
        }
    },
    "2": {
        "first_task": {
            "response": {},
            "headers": {
                "Content-Type": "application/json"
            }
        },
        "second_task": {
            "response": {},
            "headers": {
                "Content-Type": "application/json"
            }
        }
    },
    "3": {
        "first_task": {
            "response": {},
            "headers": {
                "Content-Type": "application/json"
            }
        },
        "second_task": {
            "response": {},
            "headers": {
                "Content-Type": "application/json"
            }
        }
    }
}
```

### 在循环任务中使用迭代键

有时，你可能希望在循环任务内部使用 Do While 的迭代值/计数器。在此示例中，对一个 GitHub 仓库发起 API 调用以获取所有 stargazers，每次迭代会递增分页。

要评估当前迭代，`loopCondition` 中使用参数 `$.get_all_stars_loop_ref['iteration']`。在循环内嵌的 HTTP 任务中，使用 `${get_all_stars_loop_ref.output.iteration}` 定义 API 应返回哪一页。


```json
{
    "name": "get_all_stars",
    "taskReferenceName": "get_all_stars_loop_ref",
    "inputParameters": {
        "stargazers": "4000"
    },
    "type": "DO_WHILE",
    "loopCondition": "if ($.get_all_stars_loop_ref['iteration'] < Math.ceil($.stargazers/100)) { true; } else { false; }",
    "loopOver": [
        {
            "name": "100_stargazers",
            "taskReferenceName": "hundred_stargazers_ref",
            "inputParameters": {
                "counter": "${get_all_stars_loop_ref.output.iteration}",
                "http_request": {
                    "uri": "https://api.github.com/repos/ntflix/conductor/stargazers?page=${get_all_stars_loop_ref.output.iteration}&per_page=100",
                    "method": "GET",
                    "headers": {
                        "Authorization": "token ${workflow.input.gh_token}",
                        "Accept": "application/vnd.github.v3.star+json"
                    }
                }
            },
            "type": "HTTP"
        }
    ]
}
```


## Orkes Conductor 兼容性

为兼容从 Orkes Conductor 迁移过来的工作流，`inputParameters` 中的 `_items` 参数也得到支持：

```json
{
  "name": "do_while",
  "taskReferenceName": "do_while_ref",
  "type": "DO_WHILE",
  "inputParameters": {
    "_items": "${workflow.input.myList}"
  },
  "loopOver": [...]
}
```

其行为与使用 `items` 参数完全相同。对于新工作流，推荐使用 `items` 参数。

## 局限性

Do While 任务存在以下若干局限性：

- **分支**—在 Do While 任务内部，支持使用 Switch、Fork/Join、Dynamic Fork 任务进行分支。但是，由于循环任务在 Do While 任务的作用域内执行，任何跨越其作用域之外的分支将不会生效。
- **嵌套循环**—不支持嵌套的 Do While 任务。若要实现类似嵌套循环的功能，可以在 Do While 任务内部使用 [Sub Workflow](sub-workflow-task.md) 任务。
- **隔离组执行**—不支持隔离组执行。但是，Do While 任务内部的循环任务支持 domain。
