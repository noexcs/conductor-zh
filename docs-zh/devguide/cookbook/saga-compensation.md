---
description: "Conductor 手册（cookbook）— Saga 模式示例：用 failureWorkflow 补偿部分完成的分布式事务——读取失败的执行，只按相反顺序、幂等地撤销实际运行过的步骤。"
---

# Saga：补偿部分失败

三个服务，一笔订单。库存已预留，银行卡已扣款，随后承运商返回 503。三步中已有两步完成，而没有任何事务可以回滚——每个服务各自拥有自己的数据。

本示例只撤销恰好已完成的那部分工作，按相反顺序进行，不做其他任何事情。

## 整体结构

```text
reserve_inventory ──> charge_payment ──> book_shipment
                                              │ fails
                                              ▼
                                     failureWorkflow starts
                                              │
                     read the failed execution ──> refund_payment ──> release_inventory
```

主工作流本身不包含回滚分支。它声明了一个 `failureWorkflow`，当主工作流在重试耗尽后失败时，Conductor 会启动该工作流。

## 为什么补偿必须读取失败的执行

朴素的补偿工作流会撤销每一个步骤。这是错误的：如果 `reserve_inventory` 失败了，就不存在可释放的预留，也没有可退还的扣款，盲目调用退款只会制造出一张客服工单。

Conductor 会向失败工作流传入五个输入，其中最后一个正是让此事可解的关键：

| 输入 | 它提供的内容 |
|---|---|
| `reason` | 工作流失败的原因 |
| `workflowId` | 失败执行的 id |
| `failureStatus` | 它的终态状态 |
| `failureTaskId` | 失败任务的 id |
| `failedWorkflow` | **整个失败的执行**，包括每个任务及其输出 |

因此，补偿首先向这次执行询问实际发生了什么：

```json
{
  "name": "determine_what_completed",
  "taskReferenceName": "completed_steps",
  "type": "JSON_JQ_TRANSFORM",
  "inputParameters": {
    "failed": "${workflow.input.failedWorkflow}",
    "queryExpression": "((.failed.tasks // []) | map(select(.status == \"COMPLETED\")) | map(.referenceTaskName)) as $done | {done: $done, undoPayment: ($done | index(\"charge_payment\") != null), undoInventory: ($done | index(\"reserve_inventory\") != null)}"
  }
}
```

随后，每个撤销操作都位于针对该结果的一个 `SWITCH` 之后。从未发生的事情绝不会被撤销。

## 前置条件

一个正在运行的 Conductor 服务器。本示例调用三个 HTTP 端点；附带了一个桩服务（stub），你可以无需接入真实服务即可运行。

将其保存为 `saga_stub_service.py` 并让它保持运行：

```python
--8<-- "docs/devguide/cookbook/assets/saga_stub_service.py"
```

```bash
python3 saga_stub_service.py      # http://localhost:8088
```

它会把每次调用都记录在 `GET /calls` 中，这就是你证明 Saga 实际做了什么的方式。

## 主工作流

将其保存为 `saga-order-fulfillment.json`：

```json
--8<-- "docs/devguide/cookbook/assets/saga-order-fulfillment.json"
```

## 补偿工作流

将其保存为 `saga-order-compensation.json`：

```json
--8<-- "docs/devguide/cookbook/assets/saga-order-compensation.json"
```

## 注册并运行

```bash
conductor workflow create saga-order-compensation.json
conductor workflow create saga-order-fulfillment.json
```

正常路径 —— 承运商接受发货：

```bash
conductor workflow start -w saga_order_fulfillment --sync \
  -i '{"orderId":"ORD-1","amount":49.00,"shipmentStatus":"200"}'
```

失败路径 —— 承运商已宕机，而银行卡此前已扣款：

```bash
conductor workflow start -w saga_order_fulfillment \
  -i '{"orderId":"ORD-2","amount":49.00,"shipmentStatus":"503"}'
```

在 Conductor UI 中打开 **[执行（Executions）](http://localhost:8080/executions)**，选择新的执行，查看任务图以及每个任务的输入和输出。

失败工作流的输出携带 `conductor.failure_workflow` —— 即补偿运行的 id。打开它，你会看到：

```text
completed_steps      JSON_JQ_TRANSFORM   COMPLETED
route_refund         SWITCH              COMPLETED
refund_payment       HTTP                COMPLETED
route_release        SWITCH              COMPLETED
release_inventory    HTTP                COMPLETED
```

其输出为：

```json
{
  "stepsCompleted": ["reserve_inventory", "charge_payment"],
  "paymentRefunded": true,
  "inventoryReleased": true,
  "compensatedOrder": "ORD-2"
}
```

向桩服务询问它实际收到了什么：

```bash
curl -s http://localhost:8088/calls
```

```text
1. /inventory/reserve   key=ORD-2-reserve
2. /payments/charge     key=ORD-2-charge
3. /shipping/book       key=ORD-2-ship
4. /shipping/book       key=ORD-2-ship
5. /shipping/book       key=ORD-2-ship
6. /shipping/book       key=ORD-2-ship
7. /payments/refund     key=ORD-2-refund
8. /inventory/release   key=ORD-2-release
```

有两点值得仔细端详。撤销调用是**按相反顺序**到达的——先退款、后释放。而 `/shipping/book` 在工作流放弃之前被尝试了**四次**，这正是下一节的全部论据。

## 生产环境注意事项

- **每次写入都需要幂等键。** 失败的端点会被任务重试反复调用。桩服务对重复的 `Idempotency-Key` 会重放存储的答案，而不是把活儿干两遍；你的服务必须做同样的事。
- **补偿本身也必须是幂等的。** 失败工作流本身也可能被重试。`refund_payment` 携带 `ORD-2-refund`，所以第二次尝试是空操作，而不是第二次退款。
- **只撤销已完成的部分。** 每个撤销操作都以失败执行中各任务的状态为依据，绝不以“一切都运行过”为假设。
- **给补偿比正向路径更多的重试。** 这里正向的发货调用重试一次；退款和释放则带退避地重试五次。撤销失败比执行失败更糟。
- **补偿不是回滚。** 退款是一笔新交易，有自己的账目条目。设计目标是“最终一致且可解释”，而不是“仿佛从未发生过”。
- **补偿失败时要告警。** 无法撤销的 Saga 需要人来处理。给补偿工作流配置它自己的 `failureWorkflow` 或状态监听器。
- **避免把订单 id 放进生成的状态。** 两个工作流都从调用方提供的 `orderId` 派生键，因此重启会产生相同的键。

## 相关内容

- [处理工作流错误](../how-tos/Workflows/handling-errors.md) — 重试策略、超时策略和状态监听器
- [任务超时与重试](task-timeouts-and-retries.md) — 调优正向路径
- [微服务编排](microservice-orchestration.md) — 本示例所基于的 HTTP 链
