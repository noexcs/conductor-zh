---
description: "JSON JQ Transform 任务 — 在 Conductor 工作流中使用 JQ 表达式转换和过滤 JSON 数据。"
---
# JSON JQ Transform 任务
```json
"type" : "JSON_JQ_TRANSFORM"
```

JSON JQ Transform 任务（`JSON_JQ_TRANSFORM`）使用 jq 处理 JSON 数据。它适用于将一个任务的输出转换为另一个任务的输入。

## 任务参数

在 JSON JQ Transform 任务配置中，在 `inputParameters` 内使用这些参数。


`queryExpression` 会添加到 `JSON_JQ_TRANSFORM` 的 `inputParameters` 中，与求值所需的任何其他输入值并列。

| 参数          | 类型                | 说明                                       | 必填 / 可选  |
| ------------------ | ------------------- | ------------------------------------------------- | -------------------- |
| queryExpression | String | 用于转换 JSON 数据的 jq 过滤表达式。<br/><br/> 有关构建过滤表达式的信息，请参阅 [jq 文档](https://jqlang.org/) 和 [jq 手册](https://jqlang.org/manual/)。你可以在 [jqplay.org](https://jqplay.org/) 上交互式地测试表达式。 | Required. |
| inputParameters | Map[String, Any] | 包含 jq 转换的输入。 | Required. |

## JSON 配置


以下是 JSON JQ Transform 任务的任务配置。

```json
{
  "name": "json_transform",
  "taskReferenceName": "json_transform_ref",
  "type": "JSON_JQ_TRANSFORM",
  "inputParameters": {
    "persons": [
      {
        "name": "some",
        "last": "name",
        "email": "mail@mail.com",
        "id": 1
      },
      {
        "name": "some2",
        "last": "name2",
        "email": "mail2@mail.com",
        "id": 2
      }
    ],
    "queryExpression": ".persons | map({user:{email,id}})"
  }
}
```

## 输出

JSON JQ Transform 任务将返回以下参数。

| 名称             | 类型         | 说明                                                   |
| ---------------- | ------------ | ------------------------------------------------------------- |
| result     | List[Map[String, Any]] | jq 过滤返回的 `resultList` 的第一个元素。                           |
| resultList | List[List[Map[String, Any]]] | jq 过滤返回的结果列表。                           |
| error      | String | 如果 jq 过滤失败，则为可选的错误消息。 |



## 示例

以下是使用 JSON JQ Transform 任务的一些示例。

### 简单示例

在这个示例中，jq 过滤表达式 `key3: (.key1.value1 + .key2.value2)` 将 `key1` 和 `key2` 中提供的两个字符串数组拼接为一个名为 `key3` 的数组。

```json
{
  "name": "jq_example_task",
  "taskReferenceName": "my_jq_example_task",
  "type": "JSON_JQ_TRANSFORM",
  "inputParameters": {
    "key1": {
      "value1": [
        "a",
        "b"
      ]
    },
    "key2": {
      "value2": [
        "c",
        "d"
      ]
    },
    "queryExpression": "{ key3: (.key1.value1 + .key2.value2) }"
  }
}
```

上述 JSON JQ Transform 任务将提供以下输出。在这种情况下，`resultList` 和 `result` 是相同的。

```json
{
  "result": {
    "key3": [
      "a",
      "b",
      "c",
      "d"
    ]
  },
  "resultList": [
    {
      "key3": [
        "a",
        "b",
        "c",
        "d"
      ]
    }
  ]
}
```

### 简化数据

在这个示例中，JSON JQ Transform 任务用于简化并从一个极其密集的 API 响应中提取数据。HTTP 任务从 GitHub 获取一个 stargazers（为仓库点过 star 的用户）列表，其中仅一个用户的响应如下所示：

``` json 
  
"body":[
  {
  "starred_at":"2016-12-14T19:55:46Z",
  "user":{
    "login":"lzehrung",
    "id":924226,
    "node_id":"MDQ6VXNlcjkyNDIyNg==",
    "avatar_url":"https://avatars.githubusercontent.com/u/924226?v=4",
    "gravatar_id":"",
    "url":"https://api.github.com/users/lzehrung",
    "html_url":"https://github.com/lzehrung",
    "followers_url":"https://api.github.com/users/lzehrung/followers",
    "following_url":"https://api.github.com/users/lzehrung/following{/other_user}",
    "gists_url":"https://api.github.com/users/lzehrung/gists{/gist_id}",
    "starred_url":"https://api.github.com/users/lzehrung/starred{/owner}{/repo}",
    "subscriptions_url":"https://api.github.com/users/lzehrung/subscriptions",
    "organizations_url":"https://api.github.com/users/lzehrung/orgs",
    "repos_url":"https://api.github.com/users/lzehrung/repos",
    "events_url":"https://api.github.com/users/lzehrung/events{/privacy}",
    "received_events_url":"https://api.github.com/users/lzehrung/received_events",
    "type":"User",
    "site_admin":false
  }
}
]
```

由于所需的数据只是给定日期之后（作为工作流输入 `${workflow.input.cutoff_date}` 提供）为仓库点 star 的用户的 `starred_at` 和 `login` 参数，我们可以使用 JSON JQ Transform 任务简化输出：

```json
{
  "name": "jq_cleanup_stars",
  "taskReferenceName": "jq_cleanup_stars_ref",
  "inputParameters": {
    "starlist": "${hundred_stargazers_ref.output.response.body}",
    "queryExpression": "[.starlist[] | select (.starred_at > \"${workflow.input.cutoff_date}\") |{occurred_at:.starred_at, member: {github:  .user.login}}]"
  },
  "type": "JSON_JQ_TRANSFORM",
  "decisionCases": {},
  "defaultCase": [],
  "forkTasks": [],
  "startDelay": 0,
  "joinOn": [],
  "optional": false,
  "defaultExclusiveJoinTask": [],
  "asyncComplete": false,
  "loopOver": []
}
```

在上述任务配置中，API 响应 JSON 存储在 `starlist` 参数中。`queryExpression` 读取 JSON，仅选择 `starred_at` 值满足日期条件的条目，并生成以下格式的输出 JSON：

```json
{
  "occurred_at": "date from JSON",
  "member":{
    "github" : "github Login from JSON"
  }
}
```

`queryExpression` 被包裹在 `[]` 中，表示响应应为数组。
