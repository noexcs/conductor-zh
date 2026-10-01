---
description: "了解 Conductor 中的工作者——执行工作流中任务的代码，可以用任何语言编写，托管在你选择的任何地方。"
---

# 工作者

<section class="concept-hero concept-hero--workers">
  <svg class="concept-hero__graphic" viewBox="0 28 440 145" role="img" aria-label="Conductor queues a task for a worker, which executes it and reports a result">
    <defs><marker id="worker-arrow" markerWidth="8" markerHeight="8" refX="7" refY="4" orient="auto"><path d="M0,0 L8,4 L0,8 Z" fill="currentColor" /></marker></defs>
    <rect x="14" y="68" width="102" height="54" rx="10" class="concept-hero__node concept-hero__node--accent" />
    <text x="65" y="91" text-anchor="middle" class="concept-hero__label">Conductor</text>
    <text x="65" y="108" text-anchor="middle" class="concept-hero__detail">分派</text>
    <path d="M116 95 H161" class="concept-hero__line" marker-end="url(#worker-arrow)" />
    <rect x="169" y="68" width="99" height="54" rx="10" class="concept-hero__node" />
    <text x="218" y="91" text-anchor="middle" class="concept-hero__label">任务队列</text>
    <text x="218" y="108" text-anchor="middle" class="concept-hero__detail">轮询</text>
    <path d="M268 95 H311" class="concept-hero__line" marker-end="url(#worker-arrow)" />
    <rect x="319" y="36" width="106" height="54" rx="10" class="concept-hero__node" />
    <text x="372" y="59" text-anchor="middle" class="concept-hero__label">工作者</text>
    <text x="372" y="76" text-anchor="middle" class="concept-hero__detail">执行</text>
    <path d="M372 90 V139 H268" class="concept-hero__line" marker-end="url(#worker-arrow)" />
    <rect x="169" y="123" width="99" height="42" rx="10" class="concept-hero__outcome-box" />
    <text x="218" y="149" text-anchor="middle" class="concept-hero__label">结果</text>
  </svg>
</section>

一个**工作者**负责执行工作流中的任务。每种类型的工作者实现每个任务的核心功能，处理其代码中定义的逻辑。

系统任务工作者由 Conductor 在其 JVM 内管理，而 `SIMPLE` 任务工作者由你自己实现。这些工作者可以用任何你选择的编程语言（Python、Java、JavaScript、C#、Go 和 Clojure）实现，并托管在 Conductor 环境之外的任何地方。

!!! 注意
    Conductor 在其 SDK 中提供了一组工作者框架。这些框架自带轮询线程、指标和服务器通信等功能，使创建自定义工作者变得容易。

这些工作者通过 REST/gRPC 与 Conductor 服务器通信，使其能够轮询任务并更新任务状态。详见 [架构](../architecture/index.md)。


## 工作者如何工作

1. **轮询** — 工作者从 Conductor 服务器轮询特定类型的任务。
2. **执行** — 工作者接收任务，执行业务逻辑，并产生输出。
3. **上报** — 工作者将任务结果（COMPLETED 或 FAILED）上报给服务器。

Conductor 处理调度、重试和状态持久化。你的工作者只需专注于业务逻辑。


## 工作者配置

工作者通过 Conductor 服务器上的任务定义进行配置。关键设置：

| 参数 | 描述 |
| :--- | :--- |
| `retryCount` | Conductor 重试失败任务的次数。 |
| `retryDelaySeconds` | 重试之间的延迟。 |
| `responseTimeoutSeconds` | 工作者轮询后响应的最长时间。 |
| `timeoutSeconds` | 任务完成的整体 SLA。 |
| `pollTimeoutSeconds` | 工作者轮询超时前的最长时间。 |
| `rateLimitPerFrequency` | 每个频率窗口内最大的任务执行次数。 |
| `concurrentExecLimit` | 跨所有工作者的最大并发执行数。 |

完整参考见 [任务定义](../../documentation/configuration/taskdef.md)。


## 扩展任务工作者

工作者可以独立于 Conductor 服务器进行扩展：

- **水平扩展** — 运行同一工作者的多个实例。Conductor 自动将任务分配给所有轮询的工作者。
- **速率限制** — 使用 `rateLimitPerFrequency` 控制每个任务类型的吞吐量。
- **并发限制** — 使用 `concurrentExecLimit` 限制并行执行数。
- **域隔离** — 使用 [任务域](../../documentation/api/taskdomains.md) 将任务路由到特定的工作者组。

详细指南见 [扩展工作者](../how-tos/Workers/scaling-workers.md)。
