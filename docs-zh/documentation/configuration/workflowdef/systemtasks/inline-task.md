---
description: "Inline 任务 — 在 Conductor 工作流中执行 JavaScript 表达式，用于数据转换和条件逻辑。"
---
# Inline 任务
```json
"type": "INLINE"
```

Inline 任务（`INLINE`）在 Conductor 服务器 JVM 内部执行轻量级脚本逻辑，并立即返回可接入下游任务的结果。

Inline 任务最适合小型、确定性的逻辑，例如简单的校验或计算。对于重量级、自定义的逻辑，最好改用工作者任务（`SIMPLE`）。

## 任务参数

在 Inline 任务配置中，在 `inputParameters` 内使用这些参数。

| 参数          | 类型                | 说明                                       | 必填 / 可选  |
| ------------------ | ------------------- | ------------------------------------------------- | -------------------- |
| evaluatorType | String | 使用的求值器类型。支持的类型：`graaljs`（推荐）、`javascript`、`python`、`value-param`。 | Required. |
| expression    | String | 由求值器求值的表达式。表达式必须返回一个值。<br/><br/> `graaljs` 求值器使用 GraalVM JavaScript，支持现代 ECMAScript。`python` 求值器通过 GraalVM 多语言运行 Python。`javascript` 求值器是遗留选项。`value-param` 求值器直接返回参数值。 | Required. |
| inputParameters    | Map[String, Any] | Inline 任务的其他输入参数。你可以在这里包含求值所需的任何其他输入值，它们在 `expression` 中可以通过 `$.value` 引用。 | Optional. |

## JSON 配置

以下是 Inline 任务的任务配置。

```json
{
  "name": "inline",
  "taskReferenceName": "inline_ref",
  "type": "INLINE",
  "inputParameters": {
    "evaluatorType": "javascript",
    "expression": "(function(){ return $.input1 + $.input2; })()",
    "input1": 1,
    "input2": 2
  }
}
```


## 输出

Inline 任务将返回以下参数。

| 名称             | 类型         | 说明                                                   |
| ---------------- | ------------ | ------------------------------------------------------------- |
| result | Map  | 包含求值器基于 `expression` 返回的输出。 |

## 示例

以下是使用 Inline 任务的一些示例。

### 简单示例

``` json
{
  "name": "INLINE_TASK",
  "taskReferenceName": "inline_test",
  "type": "INLINE",
  "inputParameters": {
      "inlineValue": "${workflow.input.inlineValue}",
      "evaluatorType": "javascript",
      "expression": "function scriptFun(){if ($.inlineValue == 1){ return {testvalue: true} } else { return
      {testvalue: false} }} scriptFun();"
  }
}
```

Inline 任务的输出可以在下游任务中使用表达式
`"${inline_test.output.result.testvalue}"` 引用。


### 格式化数据

在这个示例中，Inline 任务用于确保下游任务只收到以摄氏度表示的天气数据。

``` json
{
  "name": "INLINE_TASK",
  "taskReferenceName": "inline_test",
  "type": "INLINE",
  "inputParameters": {
      "scale": "${workflow.input.tempScale}",
	    "temperature": "${workflow.input.temperature}",
      "evaluatorType": "javascript",
      "expression": "function SIvaluesOnly(){if ($.scale === "F"){ centigrade = ($.temperature -32)*5/9; return {temperature: centigrade} } else { return 
      {temperature: $.temperature} }} SIvaluesOnly();"
  }
}
```
