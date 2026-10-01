---
description: 向工作流或正在运行的子工作流中第一个被阻塞的 WAIT 发送信号；如需精确指定目标任务，请使用任务更新 API。
---

# 向工作流发送信号

<section class="concept-hero concept-hero--event-bus" aria-label="Send signals">
  <div class="concept-hero__content">
    <p>一个<strong>信号</strong>推动一个已在运行并处于等待状态的工作流向前。它解决目标执行中第一个非终态的 <code>WAIT</code> 任务，因此调用方只需要工作流 ID。信号绝不会启动新的执行，无法指向任意的任务引用，也不解决 <code>HUMAN</code> 任务。</p>
  </div>
  <svg class="concept-hero__graphic event-hero__graphic" viewBox="0 0 440 190" role="img" aria-labelledby="signal-svg-title signal-svg-desc" xmlns="http://www.w3.org/2000/svg">
    <title id="signal-svg-title">工作流信号流</title>
    <desc id="signal-svg-desc">调用方向信号端点发送一个输出载荷。它找到第一个被阻塞的 WAIT（包括位于正在运行的子工作流中的），然后工作流继续。</desc>
    <defs><marker id="signal-arrow" markerWidth="8" markerHeight="8" refX="7" refY="4" orient="auto"><path d="M0,0 L8,4 L0,8 z" fill="currentColor"/></marker></defs>
    <rect x="14" y="68" width="96" height="54" rx="10" class="concept-hero__node event-hero__node--broker"/><text x="62" y="91" text-anchor="middle" class="concept-hero__label">调用方</text><text x="62" y="108" text-anchor="middle" class="concept-hero__detail">决策输出</text>
    <path d="M110 95 H145" class="concept-hero__line" marker-end="url(#signal-arrow)"/>
    <rect x="153" y="68" width="100" height="54" rx="10" class="concept-hero__node concept-hero__node--accent"/><text x="203" y="91" text-anchor="middle" class="concept-hero__label">信号 API</text><text x="203" y="108" text-anchor="middle" class="concept-hero__detail">查找第一个 WAIT</text>
    <path d="M253 95 H288" class="concept-hero__line" marker-end="url(#signal-arrow)"/>
    <rect x="296" y="51" width="130" height="88" rx="10" class="concept-hero__node event-hero__node--action"/><text x="361" y="77" text-anchor="middle" class="concept-hero__label">被阻塞的 WAIT</text><text x="361" y="94" text-anchor="middle" class="concept-hero__detail">工作流或运行中的</text><text x="361" y="111" text-anchor="middle" class="concept-hero__detail">子工作流</text><text x="361" y="128" text-anchor="middle" class="concept-hero__detail">然后继续</text>
  </svg>
</section>

## 定义一个等待信号的工作流

这个工作流记录一条审批请求，然后等待另一个系统提供决定。

```json
{
  "name": "order_approval",
  "description": "Wait for an external order approval signal",
  "version": 1,
  "schemaVersion": 2,
  "inputParameters": ["orderId"],
  "tasks": [
    {
      "name": "wait_for_approval",
      "taskReferenceName": "approval",
      "type": "WAIT"
    }
  ],
  "outputParameters": {
    "orderId": "${workflow.input.orderId}",
    "approval": "${approval.output}"
  }
}
```

通过工作流元数据 API 注册它：

```shell
curl -X POST 'http://localhost:8080/api/metadata/workflow' \
  -H 'Content-Type: application/json' \
  -d @order_approval.json
```

## 启动并等待阻塞任务

同步执行端点会启动工作流，并等待其进入终态或出现被阻塞的 `WAIT` 任务。`waitForSeconds` 默认为 `10`；当某个终态任务引用也应结束等待时，使用 `waitUntilTaskRef`。

```shell
curl -X POST 'http://localhost:8080/api/workflow/execute/order_approval/1?requestId=approval-demo-42&waitForSeconds=30&returnStrategy=BLOCKING_TASK_INPUT' \
  -H 'Content-Type: application/json' \
  -d '{"input":{"orderId":"order-42"}}'
```

`returnStrategy` 控制响应的形态：

| 值 | 返回内容 |
|---|---|
| `TARGET_WORKFLOW` | 按 ID 请求的工作流。这是默认值。 |
| `BLOCKING_WORKFLOW` | 包含当前阻塞点的工作流；它可以是子工作流。 |
| `BLOCKING_TASK` | 当前阻塞的任务。 |
| `BLOCKING_TASK_INPUT` | 当前阻塞任务的输入。 |

## 异步发送信号以解除等待

当调用方只需要提交决定时，使用异步信号端点。它完成当前被阻塞的 `WAIT` 任务并立即返回。

```shell
curl -X POST 'http://localhost:8080/api/tasks/<workflow-id>/COMPLETED/signal' \
  -H 'Content-Type: application/json' \
  -d '{"approved":true,"approvedBy":"manager@example.com","reason":"Within policy"}'
```

信号的目标是工作流中第一个非终态的 `WAIT` 任务，包括正在运行的子工作流中的任务。它不会指向 `HUMAN` 任务或任意的任务引用。信号不指定任务引用；只有当你想要的正是这种“当前阻塞等待”行为时，才应使用此端点。如果需要精确指向某个任务，请改用任务更新端点（`POST /api/tasks/{workflowId}/{taskRefName}/{status}`）。

## 发送信号并等待下一个工作流状态

当调用方需要在同一个响应中拿到结果工作流状态时，使用同步变体。它接受相同的 `returnStrategy` 值，并最多等待 `timeoutMillis`（默认：`5000`）。

```shell
curl -X POST 'http://localhost:8080/api/tasks/<workflow-id>/COMPLETED/signal/sync?returnStrategy=TARGET_WORKFLOW&timeoutMillis=5000' \
  -H 'Content-Type: application/json' \
  -d '{"approved":true,"approvedBy":"manager@example.com"}'
```

如果工作流到达了另一个 `WAIT` 任务，响应表示下一个阻塞状态；如果它先完成，响应表示已完成的工作流。当没有可发送信号的被阻塞任务时，同步信号返回 `404`；异步路由在提交信号后即返回，其响应中不提供该状态。

## 拒绝或使等待失败

从 URL 中选择任务状态来记录不同的决定。例如，当审批被拒绝且你希望工作流的失败路径运行时，发送 `FAILED` 信号：

```shell
curl -X POST 'http://localhost:8080/api/tasks/<workflow-id>/FAILED/signal' \
  -H 'Content-Type: application/json' \
  -d '{"reason":"Order exceeds the approval limit"}'
```

你发送的载荷会被存储为该 `WAIT` 任务的输出。下游任务可以使用 `${approval.output.approved}` 或 `${approval.output.reason}` 之类的表达式引用它。

## 后续步骤

<div class="event-next-steps">
  <a href="../how-tos/consume-route-events.html">将 broker 事件路由到任务 →</a>
  <a href="../how-tos/incoming-webhooks.html">接收已验证的 webhook →</a>
  <a href="../how-tos/event-bus.html">事件驱动概览 →</a>
  <a href="wait-and-timers.html">等待与定时器模式 →</a>
</div>
