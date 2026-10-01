---
description: "以名称和版本将 JSON Schema 存储在 Conductor 服务器上，让许多定义引用同一份契约，而不是各自内嵌一份拷贝。"
---

# Schema 注册表

附着到定义上的 Schema 随该定义一起走——参见[输入/输出 Schema 校验](schema-validation.md)。这行得通，但一旦两个定义需要同一个契约，它就不行了：你有了两份会逐渐漂移的拷贝。

Schema 注册表是解决这个问题的服务器端存储。一个 Schema 以一个名称和一个版本保存一次，定义引用它。UI 中输入和输出 Schema 选择器展示的内容也来自这里。

## API

`/api/schema` 之下有六个端点。路径、方法和参数与 Conductor SDK 所依据的契约一致，所以现有的 Schema 客户端无需任何改动即可与此服务器通信。

| 方法 | 路径 | 用途 |
|---|---|---|
| `POST` | `/api/schema?newVersion=false` | 保存一个或多个 Schema |
| `GET` | `/api/schema?short=false` | 列出每个 Schema 的所有版本 |
| `GET` | `/api/schema/{name}` | 读取某名称下注册的版本号最高的版本 |
| `GET` | `/api/schema/{name}/{version}` | 读取一个版本 |
| `DELETE` | `/api/schema/{name}` | 删除某名称下的所有版本 |
| `DELETE` | `/api/schema/{name}/{version}` | 删除一个版本 |

### 保存

请求体是一个**列表**，`POST` 返回 `200` 且无响应体。

```shell
curl -X POST "$CONDUCTOR_SERVER_URL/schema" \
  -H 'Content-Type: application/json' \
  -d '[{
    "name": "customerInput",
    "version": 1,
    "type": "JSON",
    "data": {
      "$schema": "https://json-schema.org/draft/2020-12/schema",
      "type": "object",
      "properties": { "customerId": { "type": "string" } },
      "required": ["customerId"]
    }
  }]'
```

裸对象也被接受，并被当作单元素列表处理。一些 SDK 客户端就是发送裸对象的，所以这不是你需要避免的简写。

### 读取

```shell
curl "$CONDUCTOR_SERVER_URL/schema/customerInput"
```

```json
{
    "createTime": 1788197572423,
    "updateTime": 0,
    "name": "customerInput",
    "version": 1,
    "type": "JSON",
    "data": {
        "$schema": "https://json-schema.org/draft/2020-12/schema",
        "type": "object",
        "properties": {
            "customerId": {
                "type": "string"
            }
        },
        "required": [
            "customerId"
        ]
    }
}
```

`GET /api/schema/{name}/{version}` 读取一个指定版本而非最新版本。两者在没有注册任何内容时都返回 `404`，这就是你区分"Schema 不存在"和"Schema 为空"的方式：

```json
{"status":404,"message":"No such schema found by name customerInput","instance":"5f0694ae4d22","retryable":false}
```

### 列表

`GET /api/schema` 返回每个 Schema 的所有版本，包含文档体。`?short=true` 只返回名称和版本——这是选择器所需要的，这样打开下拉框不会传输服务器上的每个 Schema 文档：

```shell
curl "$CONDUCTOR_SERVER_URL/schema?short=true"
```

```json
[
    {
        "createTime": 0,
        "updateTime": 0,
        "name": "customerInput",
        "version": 1
    },
    {
        "createTime": 0,
        "updateTime": 0,
        "name": "customerInput",
        "version": 2
    }
]
```

清零的时间戳是占位符，不是真实的创建日期——短列表连同 Schema 文档体一起省略了它们。需要时请读取完整记录。

## 版本管理

Schema 以名称**和**版本寻址；这个组合是唯一的。`version` 默认为 `1`。

有两种保存方式，差异很重要：

| `newVersion` | 效果 |
|---|---|
| `false`（默认） | 覆盖载荷中该版本上已存储的任何内容 |
| `true` | 存储到该名称下当前已注册的最高版本加一 |

用 `newVersion=true` 来演进契约。固定到旧版本的定义会继续解析到它们所依据的 Schema：

```shell
curl -X POST "$CONDUCTOR_SERVER_URL/schema?newVersion=true" \
  -H 'Content-Type: application/json' \
  -d '[{ "name": "customerInput", "type": "JSON", "data": { "...": "..." } }]'
```

```shell
curl "$CONDUCTOR_SERVER_URL/schema/customerInput"   # now version 2
curl "$CONDUCTOR_SERVER_URL/schema/customerInput/1" # still the original
```

用默认方式就地修正某个版本——描述里的拼写错误、一个本应设为可选的字段。引用该版本的任何东西都会看到修正，这正是目的，也是风险所在。

对同一名称同时进行两次 `newVersion=true` 保存可能互相覆盖。服务器读取最高版本然后保存其加一，两步之间没有任何隔离，所以两个写入者可能读到相同的最大值、落到同一个版本，最终只留下后写入的那份。同一名称下的并发注册是后写者胜出；如果丢失某一份会有影响，就串行化这些保存。

删除同样感知版本。`DELETE /api/schema/{name}/{version}` 删除一个版本并保留其余历史；`DELETE /api/schema/{name}` 删除全部。两者在没有任何可删内容时都返回 `404`，所以一个返回 `200` 的删除确实删除了东西——如果你编写无论 Schema 是否存在都会运行的清理脚本，知道这点很有用。

## 管理界面

UI 有一个针对注册表的界面，所以日常工作不需要 `curl`。在 **Definitions → Schemas** 下找到它，路径为 `/schemas`。

列表每个 Schema 占一行，而不是每个版本占一行：名称链接到编辑器，行上显示 Schema 的类型、最新版本、存在多少版本以及创建时间。每行有两个操作——**Clone**，它以新名称复制该契约并从版本 1 重新开始；**Delete**，它删除该 Schema 及其所有版本。

打开一个 Schema 会在 JSON 编辑器中显示其文档体，并带有一个版本选择器供查看其历史。从那里可以：

| 操作 | 作用 |
|---|---|
| **Save** | 覆盖屏幕上的版本。引用该版本的任何东西都会看到变更，所以此操作会先请求确认 |
| **Save as new version** | 将编辑后的文档体存储为新版本。服务器分配版本号，所以两个人同时保存不会冲突 |
| **Delete version** | 删除屏幕上的版本并保留其余历史 |
| **Reset** | 丢弃本地编辑并重新加载已存储的版本 |
| **Download** | 把屏幕上的 Schema 保存为 `.json` 文件 |

**New schema**（新建 Schema）在一个 JSON 模板上打开同一个编辑器。保存它即注册版本 1。

编辑器只能编写 `JSON` Schema。已存储的 `AVRO` 或 `PROTOBUF` Schema 以只读方式打开，并附有说明：此服务器不校验它——界面不会让你编辑一个其类型在此没有任何东西强制执行的 Schema。要通过 API 替换它们。

Simple Task、Yield Task、Workflow Properties 和 Task Definition 表单上的输入和输出 Schema 选择器读取同一个注册表，只要服务器提供 `/api/schema` 就会填充。在 Simple Task、Yield Task 和 Workflow Properties 表单上，指定了注册表不持有的 Schema 的选择器会被标记，这样悬空引用会在编辑器中暴露，而不是在运行时。Task Definition 表单不做此标记。

给每个 JSON Schema 都加一行 `$schema`，如上例所示。没有它，服务器无法判断应套用哪个 JSON Schema 版本，而强制执行该 Schema 的定义会静默地什么都不校验。参见[输入/输出 Schema 校验](schema-validation.md)。

## 服务器属性

注册表本身不需要配置，强制执行也不需要：某个定义的 Schema 是否被强制执行由该定义自身的 `enforceSchema` 标志决定，而不是由服务器设置决定。参见[输入/输出 Schema 校验](schema-validation.md)。缓存是唯一可在此配置的东西，且默认关闭。

| 属性 | 默认值 | 含义 |
|---|---|---|
| `conductor.app.schema-cache.ttl` | `0` | 一次读取在缓存中保留多久。零表示禁用缓存；没有单独的开/关标志 |
| `conductor.app.schema-cache.max-size` | `1000` | 缓存条目上限，按版本查找和按名称查最新两类查找分别计数 |

非零的 `ttl` 也是你的陈旧度上界。保存和删除时的失效只到达处理写入的那个节点，所以在多节点部署中，其他每个节点会一直提供旧 Schema，直到条目过期。把它设成一个你在编辑之后愿意等待到底的值。

## 存储

Schema 持久化在 MySQL、PostgreSQL、SQLite 和 Redis 上，存于 `meta_schema_def` 表（或在 Redis 上按 Schema 名称存为哈希）。SQL 后端通过注册表自己的迁移来创建它，与主 Conductor 迁移相互独立。

没有 Cassandra 实现。配置了 `conductor.db.type=cassandra` 的服务器会在启动时失败，而不是接受它无法存储的 Schema 写入。

## 限制

有四件事无法从 API 推断出来：

**三种 Schema 类型都会被存储；只有 `JSON` 会被校验。** 你可以保存 `AVRO` 或 `PROTOBUF` Schema 并原样读回，但此服务器上没有任何东西对照它校验载荷，管理界面也因此把它显示为只读。

**`createdBy` 和 `updatedBy` 永远不会被填充。** API 不带认证，所以没有主体（principal）可以归属一次写入，这些字段在响应中是缺失而不是为空。`createTime` 和 `updateTime` 正常设置。

**选择器上的内联编辑和预览按钮不会显示。** 在任务、工作流和任务定义表单的 Schema 选择器中，那些不离开表单就打开 Schema 进行编辑或预览的按钮来自一个 UI 插件，而此构建没有注册任何插件。选择已有 Schema 可以正常工作；创建和编辑在管理界面或通过这个 API 完成。

**`externalRef` 被存储并返回，没有任何东西解析它。** 如果你保存一个只带 `externalRef` 的 Schema，你会原样收到该字段——服务器不会去获取它指向的内容。

## 相关页面

- [输入/输出 Schema 校验](schema-validation.md) — 把 Schema 附加到定义
- [任务定义参考](../../documentation/configuration/taskdef.md)
- [工作流定义参考](../../documentation/configuration/workflowdef/index.md)
