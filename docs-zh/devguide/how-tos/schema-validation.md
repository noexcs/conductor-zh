---
description: "将 JSON Schema 附加到工作流定义和任务定义上，让格式错误的输入在边界处就被拒绝，而不是几个任务之后才失败。"
---

# 输入/输出 Schema 校验

Schema 是对跨越边界的数据形状的一份契约。没有它，缺失字段会被第一个解引用它的任务发现——通常是在几个任务之后，以工作者中的 `NullPointerException` 或静默为 null 的 `${...}` 表达式的形式出现。有了它，并且[开启了强制执行](#turning-enforcement-on)后，执行会在任何副作用发生之前就在边界处被拒绝。

## Schema 附着在哪里

| 附着点 | 作用范围 | 字段 |
|---|---|---|
| 工作流定义 | 工作流自身的输入和输出 | `WorkflowDef.inputSchema` / `outputSchema` |
| 任务定义 | 该任务在每个工作流中的每次使用 | `TaskDef.inputSchema` / `outputSchema` |

当契约属于任务本身时，把 Schema 放在任务定义上——每个调用 `charge_payment` 的工作流都应就支付请求的样子达成一致。当契约属于入口点时，把它放在工作流定义上，凡是由外部调用方触发的一切都属于这种情况。

## Schema 的形状

Schema 是一个 `SchemaDef`，内嵌在定义中：

```json
{
  "name": "customerInput",
  "version": 1,
  "type": "JSON",
  "data": {
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "type": "object",
    "properties": {
      "customerId": { "type": "string" },
      "tier": { "type": "string", "enum": ["standard", "premium"] }
    },
    "required": ["customerId"],
    "additionalProperties": false
  }
}
```

| 字段 | 含义 |
|---|---|
| `name` | Schema 的标识符 |
| `version` | 对照哪个已注册版本进行校验。省略则跟随注册表的最新版本；指定一个则将其固定。对于内联 `data` Schema（即文档本身），此字段被忽略 |
| `type` | `JSON`、`AVRO` 或 `PROTOBUF` |
| `data` | Schema 文档本身 |
| `externalRef` | 存放于 Conductor 之外的 Schema 的名称。原样存储并返回；**没有任何东西会解引用它**，因此它不是内联 `data` 的替代方案 |

## 将其附加到工作流

```json
{
  "name": "order_fulfillment",
  "version": 1,
  "ownerEmail": "team@example.com",
  "schemaVersion": 2,
  "enforceSchema": true,
  "inputSchema": {
    "name": "customerInput",
    "version": 1,
    "type": "JSON",
    "data": {
      "$schema": "https://json-schema.org/draft/2020-12/schema",
      "type": "object",
      "properties": { "customerId": { "type": "string" } },
      "required": ["customerId"],
      "additionalProperties": false
    }
  },
  "tasks": []
}
```

有两个字段很容易混淆。`schemaVersion` 与这些无关——它是工作流定义的*格式*版本，应为 `2`。`enforceSchema` 是逐定义的开关，而且是整个决策的全部：不存在服务器级别的设置。在 `WorkflowDef` 上它默认为 `true`，所以声明了 `inputSchema` 的工作流定义会被校验，除非你显式将其设为 `false`。在 `TaskDef` 上它默认为 `false`，所以任务需要选择加入。

按通常方式注册——Schema 随定义一起走，所以没有单独的步骤：

```shell
curl -X PUT "$CONDUCTOR_SERVER_URL/metadata/workflow" \
  -H 'Content-Type: application/json' \
  -d '[ ... definition above ... ]'
```

## 引用已注册的 Schema

定义也可以不内联 `data`，而是指定一个存放在 [Schema 注册表](schema-registry.md) 中的 Schema。是否同时指定 `version` 决定了该定义是跟随注册表还是固定到某个文档：

```json
"inputSchema": { "name": "customerInput", "type": "JSON" }
```

省略 `version` **意味着跟随注册表的最新版本**。注册一个新版本后，该定义在下次执行时就会对照它校验，无需编辑定义。对于你会演进的契约，这正是你想要的；而如果新版本不应改变现有工作流的行为，则不想要这样。

```json
"inputSchema": { "name": "customerInput", "type": "JSON", "version": 2 }
```

指定 `version` **会固定到该文档**。之后的版本被忽略；该定义会一直对照版本 2 校验，直到你修改定义。当一个定义已用真实流量检验过、不应在你脚下悄悄变化时，就固定它。

失败消息会指明实际应用的版本，而不是请求的版本，因此当某个引用拒绝了一个载荷时，固定引用与跟随引用是可区分的。

## 开启强制执行 { #turning-enforcement-on }

强制执行完全由定义决定。没有要设置的服务器属性，也没有要重启的东西。当以下两者同时成立时，载荷会被检查：

1. 定义自身的 `enforceSchema` 为 `true`；
2. 该位置确实附着了 Schema。

两个默认值不同，而当你附着 Schema 时这个差异很重要：

- 在 **`TaskDef`** 上，`enforceSchema` 默认为 `false`。直到你在同一份定义上设置该标志，附着 Schema 不会改变任务的任何执行行为，因此可以仅为文档目的附着 Schema 而不拒绝任何工作。
- 在 **`WorkflowDef`** 上，它默认为 `true`。因此仅附着 `inputSchema` 或 `outputSchema` 就足够了：该定义从下次执行开始被校验。如果你希望记录 Schema 但不强制执行，请显式将 `enforceSchema` 设为 `false`。

无论如何，强制执行都是一份定义一份定义地到来——随着你逐个编辑它们——而不是在整个部署中一次性到来。

!!! warning "升级已附着 Schema 的服务器"
    由于 `enforceSchema` 在 `WorkflowDef` 上默认为 `true`，已经携带 `inputSchema` 或 `outputSchema` 的工作流定义——在该服务器能够强制执行任何东西之前就附着上去的——在升级后的下次执行时就会开始被校验，无需编辑定义。一个仅为文档目的编写、从未用真实流量检验过的 Schema 会变成一道关卡。

    升级之前，列出会受影响的定义并逐个做出决定：

    ```shell
    curl -s "$CONDUCTOR_SERVER_URL/metadata/workflow" \
      | jq -r '.[] | select((.inputSchema != null or .outputSchema != null) and .enforceSchema != false)
               | "\(.name) v\(.version)"'
    ```

    对任何你尚未准备好强制执行的定义显式设置 `enforceSchema` 为 `false`；无论如何 Schema 都会保持记录状态。`TaskDef` 不需要这样的审查——它默认为 `false`，所以附着的任务 Schema 保持不生效，直到你选择加入。

其推论是，设置 `enforceSchema` 在该定义的下次执行时生效。在一个你尚未用真实流量检验过 Schema 的定义上设置它，该定义会立即开始拒绝载荷，所以要把它当作它本来就是的那种变更来对待：先注册 Schema，确认它与调用方实际发送的内容一致，然后再打开这个标志。

## 校验在何时运行

| 时点 | 失败时的效果 |
|---|---|
| 工作流输入 | 工作流不会启动，什么都不会创建 |
| 任务输入 | 任务在工作者看到它之前就终态失败 |
| 任务输出 | 工作者返回后任务终态失败 |
| 工作流输出 | 工作流在完成时失败而不是完成 |

工作流输入失败会报告给调用方：启动请求被以 `400` 拒绝，响应体中带有校验消息，且不会创建任何执行。其他三种情况发生在运行中的执行内部，所以校验消息会成为任务或工作流上的 `reasonForIncompletion`——在 UI 和 API 中可见，无需阅读服务器日志。

这两种任务失败都是**终态**的，不可重试。违反 Schema 的输入在下次尝试时会以相同方式违反它，而被定义拒绝的输出在任务重新运行时也是同样的形状——因此两种情况下重试都毫无作用，只会把任务的重试预算花在同一结果上。工作流级别的失败会结束执行，所以也不存在重试。

输入校验是有价值的那个：它在任何任务运行之前就拒绝执行，所以没有任何东西需要补偿。输出校验能捕捉工作者返回了错误形状的情况，否则它会以远离其起因的下游失败的形式暴露出来。

## 限制，以及触及限制时会发生什么

**外部化的输出不会被检查。** 通过外部载荷存储返回输出的工作者交给服务器的是一个存储路径而非载荷本身，所以手头没有可供校验的东西，检查被跳过。任务输入、工作流输入和工作流输出不受影响；输出小到可以内联传输的任务也不受影响。

**部分系统任务的输出会被检查，部分不会。** 在其 `execute(...)` 步骤内完成的同步系统任务——`INLINE`、`SET_VARIABLE` 之类——其输出会在决策器（decider）中校验，并像其他任务一样终态失败。两类不在覆盖范围内：如 `HTTP` 或 `SUB_WORKFLOW` 这样的异步系统任务，以及一种在调度期间就完成的同步任务。对后者，任务定义上的 `outputSchema` 会被存储但从不强制执行。这与上述外部化输出一样静默通过，下面描述的无法解析的引用也一样；而非 `JSON` 的 Schema 和无类型 Schema 则会大声失败。每个系统任务的*输入*都像其他任务一样在调度时校验，工作流输入和工作流输出不受影响。

**非 `JSON` 的 Schema 会被拒绝，而不是被跳过。** `AVRO` 或 `PROTOBUF` Schema 在注册时被接受并原样返回，但此服务器没有它的校验器——所以与其让载荷未经检查通过，不如让执行失败并说明原因。一个既附着了此类 Schema 又选择加入强制执行的定义，在你打开强制执行后会开始失败。

**不带 `type` 的 Schema，或只带 `externalRef` 的 Schema，同样会被拒绝。** 两者都没有指定此服务器可以对照检查的文档——没有任何东西会解引用 `externalRef`——所以两者都会让执行失败并说明情况。

**注册表不持有的引用会静默地停止强制执行。** 以名称和版本附着的 Schema 会在 [Schema 注册表](schema-registry.md) 中查找；如果该名称和版本下没有注册任何内容，就没有可供校验的文档，所以载荷未经检查通过而不是失败。未命中会使 `schema_registry_miss` 计数器递增，并带上 Schema 名称标签——该计数器是唯一的信号，所以如果你依赖强制执行，就要盯着它。一个已注册但校验器无法读取或使用的文档行为相同，且会被记录日志。常见原因是缺少 `$schema` 行：没有它，校验器无法判断应套用哪个 JSON Schema 版本，所以一份看起来其他方面都正确的文档实际上什么都没强制执行。两者都是定义中的错误而非载荷中的错误，这就是为什么两者都不会算到调用方头上；代价是一个指向空处的引用什么都不会强制执行。

## 编写能经受时间考验的 Schema

`additionalProperties: false` 值得多想片刻。它会把意外的字段变成硬性失败——对于你从头到尾完全掌控的契约这是对的，对于调用方合法地传递额外上下文的契约则是麻烦。除非你是认真的，否则别加。

向 `required` 添加字段对每个现有调用方都是破坏性变更。因为运行中的执行会保留它启动时的定义版本，安全路径与其他任何定义变更相同：注册一个新版本，而不是编辑当前版本。参见[管理工作流版本](Workflows/versioning-workflows.md)。

保持 Schema 精简。一个复述每个可选字段的 Schema 会变成没人更新的东西，而过时的契约比没有契约更糟。只校验那些缺失时真正会弄坏工作流的字段。

## 相关页面

- [Schema 注册表](schema-registry.md) — 以名称和版本在服务器上存储 Schema
- [任务定义参考](../../documentation/configuration/taskdef.md)
- [工作流定义参考](../../documentation/configuration/workflowdef/index.md)
- [任务输入](Tasks/task-inputs.md)
- [管理工作流版本](Workflows/versioning-workflows.md)
- [CI/CD 集成](cicd-integration.md) — 在定义到达生产环境之前验证它们
