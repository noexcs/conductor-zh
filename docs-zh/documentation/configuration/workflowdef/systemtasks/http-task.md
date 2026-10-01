---
description: "在 Conductor 中配置 HTTP 任务以调用远程 API 和服务。支持带请求头、请求体和超时选项的 GET、POST、PUT、DELETE 方法。"
---

# HTTP 任务

```json
"type" : "HTTP"
```

HTTP 任务（`HTTP`）适用于调用通过 HTTP/HTTPS 暴露的远程服务。它支持调用 API 或与远程服务交互所需的多种 HTTP 方法、请求头、请求体内容以及其他配置。

HTTP 调用返回的数据可以作为输入在后续任务中引用，从而让你能够串联多个任务或 HTTP 调用来创建复杂流程，而无需编写任何额外代码。


## 任务参数

HTTP 请求参数可以直接指定在 `inputParameters` 中，也可以嵌套在 `inputParameters.http_request` 内。两种形式都支持 — 扁平形式对大多数用例更简单。

| 参数          | 类型                | 说明                                       | 必填 / 可选  |
| ------------------ | ------------------- | ------------------------------------------------- | -------------------- |
| uri | String        | HTTP 服务的 URI。支持 `${workflow.input.url}` 之类的动态引用。                                  | Required. |
| method            | String           | HTTP 方法。支持的方法：`GET`、`PUT`、`POST`、`PATCH`、`DELETE`、`OPTIONS`、`HEAD`、`TRACE`。                                       | Required. |
| accept            | String           | 服务器要求的 accept 请求头。默认：`application/json`。                                     | Optional. |
| contentType       | String           | 请求的内容类型。默认：`application/json`。                                                              | Optional. |
| headers           | Map[String, Any] | 要随请求一起发送的额外 HTTP 请求头映射。见下文 [Sending headers](#sending-headers)。                | Optional. |
| body              | Map[String, Any]            | 请求体。                                          | POST、PUT 或 PATCH 方法必填。 |
| asyncComplete     | Boolean          | 任务是否异步完成。默认：`false`。当为 `true` 时，任务保持 `IN_PROGRESS` 状态，直到外部事件将其标记为完成。 | Optional. |
| connectionTimeOut | Integer          | 连接超时时间（毫秒）。默认：100。设为 0 表示不超时。                       | Optional. |
| readTimeOut       | Integer          | 读取超时时间（毫秒）。默认：150。设为 0 表示不超时。                       | Optional. |

## 配置 JSON

以下是 HTTP 任务的任务配置。注意参数直接指定在 `inputParameters` 中：

```json
{
  "name": "http",
  "taskReferenceName": "http_ref",
  "type": "HTTP",
  "inputParameters": {
    "uri": "https://api.example.com/data",
    "method": "POST",
    "headers": {
      "Authorization": "Bearer ${workflow.input.api_token}",
      "X-Request-Id": "${workflow.correlationId}"
    },
    "body": {
      "key": "value"
    }
  }
}
```

!!! note "传统 `http_request` 形式"
    出于向后兼容，嵌套的 `inputParameters.http_request` 形式仍然受支持：
    ```json
    "inputParameters": {
      "http_request": {
        "uri": "https://api.example.com/data",
        "method": "POST",
        "body": { "key": "value" }
      }
    }
    ```
    两种形式行为完全一致。新工作流推荐使用扁平形式（如上所示）。

## 发送请求头 { #sending-headers }

使用 `headers` 参数发送自定义 HTTP 请求头，包括认证：

### Bearer 令牌认证

```json
{
  "name": "call_api",
  "taskReferenceName": "call_api_ref",
  "type": "HTTP",
  "inputParameters": {
    "uri": "https://api.example.com/protected/resource",
    "method": "GET",
    "headers": {
      "Authorization": "Bearer ${workflow.input.access_token}"
    }
  }
}
```

### API 密钥认证

```json
{
  "name": "call_api",
  "taskReferenceName": "call_api_ref",
  "type": "HTTP",
  "inputParameters": {
    "uri": "https://api.example.com/data",
    "method": "GET",
    "headers": {
      "X-API-Key": "${workflow.input.api_key}"
    }
  }
}
```

### Basic 认证

```json
{
  "name": "call_api",
  "taskReferenceName": "call_api_ref",
  "type": "HTTP",
  "inputParameters": {
    "uri": "https://api.example.com/data",
    "method": "GET",
    "headers": {
      "Authorization": "Basic ${workflow.input.basic_auth_token}"
    }
  }
}
```

### 多个自定义请求头

```json
{
  "name": "call_api",
  "taskReferenceName": "call_api_ref",
  "type": "HTTP",
  "inputParameters": {
    "uri": "https://api.example.com/data",
    "method": "POST",
    "headers": {
      "Authorization": "Bearer ${workflow.input.token}",
      "X-Correlation-Id": "${workflow.correlationId}",
      "X-Request-Source": "conductor",
      "Accept-Language": "en-US"
    },
    "body": {
      "data": "${workflow.input.payload}"
    }
  }
}
```

## 输出

HTTP 任务将返回以下参数。

| 名称   | 类型 | 说明                                                                                               |
| ------ | ---- | --------------------------------------------------------------------------------------------------------- |
| response     | Map[String, Any]              | 如果可用，包含请求响应的 JSON 体。                         |
| response.headers      | Map[String, Any] | 响应请求头。                                                            |
| response.statusCode   | Integer          | 指示请求结果的 [HTTP 状态码](https://en.wikipedia.org/wiki/List_of_HTTP_status_codes)。 |
| response.reasonPhrase | String           | 与 HTTP 状态码关联的原因短语。                                            |
| response.body | Map[String, Any] | 包含端点返回数据的响应体。

## 执行

当远程服务成功响应后，HTTP 任务会移入 COMPLETED 状态。

如果你的 HTTP 任务没有被执行，可能是任务队列中积压了过多的 HTTP 任务。考虑使用 Isolation Groups 将某些 HTTP 任务优先于其他任务执行。

## 示例

以下是使用 HTTP 任务的一些示例。


### GET 方法

```json
{
  "name": "Get Example",
  "taskReferenceName": "get_example",
  "type": "HTTP",
  "inputParameters": {
    "uri": "https://jsonplaceholder.typicode.com/posts/${workflow.input.queryid}",
    "method": "GET"
  }
}
```

### POST 方法

```json
{
  "name": "http_post_example",
  "taskReferenceName": "post_example",
  "type": "HTTP",
  "inputParameters": {
    "uri": "https://jsonplaceholder.typicode.com/posts/",
    "method": "POST",
    "body": {
      "title": "${get_example.output.response.body.title}",
      "userId": "${get_example.output.response.body.userId}",
      "action": "doSomething"
    }
  }
}
```

### PUT 方法
```json
{
  "name": "http_put_example",
  "taskReferenceName": "put_example",
  "type": "HTTP",
  "inputParameters": {
    "uri": "https://jsonplaceholder.typicode.com/posts/1",
    "method": "PUT",
    "body": {
      "title": "${get_example.output.response.body.title}",
      "userId": "${get_example.output.response.body.userId}",
      "action": "doSomethingDifferent"
    }
  }
}
```

### DELETE 方法
```json
{
  "name": "DELETE Example",
  "taskReferenceName": "delete_example",
  "type": "HTTP",
  "inputParameters": {
    "uri": "https://jsonplaceholder.typicode.com/posts/1",
    "method": "DELETE"
  }
}
```
