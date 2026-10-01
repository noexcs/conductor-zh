---
description: Conductor 编排子 agent 的九种方式、何时选用每一种，以及每个 SDK 中的可运行示例。
---

# Multi-Agent 架构

一个多 agent 系统是一个父 agent，加上一组子 agent，以及一个决定它们如何运行的 **strategy（策略）**。策略只是单个字段。其余的一切 — 持久化、重试、每次委派的可观测性 — 都来自 Conductor 把整个东西编译成一个工作流。

```python
support = Agent(
    name="support_supervisor",
    model="openai/gpt-4o-mini",
    instructions="Route each request to the right specialist.",
    agents=[billing, technical, sales],
    strategy=Strategy.HANDOFF,
)
```

## 选择策略

关键问题是**由谁决定**：模型、图，还是你。

| 策略 | 谁决定 | 运行 | 何时使用 |
|---|---|---|---|
| `handoff` | 模型 | 一个子 agent，以对话方式 | 应该由专家接管对话时 |
| `router` | 模型 | 一个子 agent，无对话 | 只需要分类和分发时 |
| `sequential` | 图 | 全部，按顺序 | 每一步都基于前一步的输出时 |
| `parallel` | 图 | 全部，同时 | 想要比较的独立意见时 |
| `swarm` | 子 agent 之间 | 直到某一个完成 | agent 应彼此传递控制权时 |
| `round_robin` | 图 | 轮转中的下一个 | 分摊负载或轮换审阅者时 |
| `random` | 图 | 随机一个 | agent 版本之间的 A/B 对比时 |
| `plan_execute` | 先模型，后图 | 一个计划的序列，随执行重新规划 | 步骤无法事先确定时 |
| `manual` | 你，在代码中 | 你选择的任何 agent | 路由是业务规则而非判断时 |

两条实用提示。**`router` 比 `handoff` 便宜** — 它分类并分发，而不接管对话，所以没有可对话内容时使用它。另外 **`plan_execute` 是唯一会重新规划的策略**；其他策略一旦做出分发决定就不再更改。

## 各种形态

=== "模型选一个"

    `handoff` 和 `router`。子 agent 作为可调用的工具暴露给父 agent 的模型。

    ```python
    support = Agent(
        name="support",
        model=MODEL,
        instructions="Route to billing, technical, or sales.",
        agents=[billing, technical, sales],
        strategy=Strategy.HANDOFF,   # or Strategy.ROUTER
    )
    ```

=== "图运行全部"

    `sequential` 和 `parallel`。模型不参与决定顺序。

    ```python
    pipeline = Agent(
        name="review_pipeline",
        model=MODEL,
        agents=[researcher, writer, editor],
        strategy=Strategy.SEQUENTIAL,   # or Strategy.PARALLEL
    )
    ```

=== "agent 之间互相交接"

    `swarm`。控制权在子 agent 之间传递，直到某一个给出最终答案。

    ```python
    swarm = Agent(
        name="triage_swarm",
        model=MODEL,
        agents=[intake, diagnosis, resolution],
        strategy=Strategy.SWARM,
    )
    ```

=== "规划、执行、重规划"

    `plan_execute`。模型生成一个子 agent 调用计划，执行它，并在结果返回时修正计划。

    ```python
    planner = Agent(
        name="incident_planner",
        model=MODEL,
        agents=[log_reader, metrics_reader, remediation_drafter],
        strategy=Strategy.PLAN_EXECUTE,
    )
    ```

## Conductor 增加了什么

- **每次委派都是独立的执行。** 专家 agent 可以重试，而无需重新运行路由决定。
- **选择被记录下来。** 哪个子 agent 运行了、为什么，都在执行记录里 — 不只是日志里的一行。
- **子 agent 保留各自自己的工具和护栏，** 因此计费 agent 够不到履约工具。
- **并行就是真正的并行。** `parallel` 和扇出会编译为 `FORK_JOIN`，而不是循环。

## 每个 SDK 中的可运行示例

下面的每种策略都在全部四个 SDK 中针对 `main` 分支验证过。

| 策略 | Python | Java | TypeScript | C# |
|---|---|---|---|---|
| `handoff` | [`05_handoffs.py`](https://github.com/conductor-oss/python-sdk/blob/main/examples/agents/05_handoffs.py) | [`Example05Handoffs.java`](https://github.com/conductor-oss/java-sdk/blob/main/agent-examples/src/main/java/org/conductoross/conductor/ai/examples/Example05Handoffs.java) | [`05-handoffs.ts`](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/agents/05-handoffs.ts) | [`05_Handoffs`](https://github.com/conductor-oss/csharp-sdk/blob/main/Conductor.AI.Examples/05_Handoffs/Program.cs) |
| `router` | [`08_router_agent.py`](https://github.com/conductor-oss/python-sdk/blob/main/examples/agents/08_router_agent.py) | [`Example08RouterAgent.java`](https://github.com/conductor-oss/java-sdk/blob/main/agent-examples/src/main/java/org/conductoross/conductor/ai/examples/Example08RouterAgent.java) | [`08-router-agent.ts`](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/agents/08-router-agent.ts) | [`08_RouterAgent`](https://github.com/conductor-oss/csharp-sdk/blob/main/Conductor.AI.Examples/08_RouterAgent/Program.cs) |
| `sequential` | [`06_sequential_pipeline.py`](https://github.com/conductor-oss/python-sdk/blob/main/examples/agents/06_sequential_pipeline.py) | [`Example06SequentialPipeline.java`](https://github.com/conductor-oss/java-sdk/blob/main/agent-examples/src/main/java/org/conductoross/conductor/ai/examples/Example06SequentialPipeline.java) | [`06-sequential-pipeline.ts`](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/agents/06-sequential-pipeline.ts) | [`06_SequentialPipeline`](https://github.com/conductor-oss/csharp-sdk/blob/main/Conductor.AI.Examples/06_SequentialPipeline/Program.cs) |
| `parallel` | [`07_parallel_agents.py`](https://github.com/conductor-oss/python-sdk/blob/main/examples/agents/07_parallel_agents.py) | [`Example07ParallelAgents.java`](https://github.com/conductor-oss/java-sdk/blob/main/agent-examples/src/main/java/org/conductoross/conductor/ai/examples/Example07ParallelAgents.java) | [`07-parallel-agents.ts`](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/agents/07-parallel-agents.ts) | [`07_ParallelAgents`](https://github.com/conductor-oss/csharp-sdk/blob/main/Conductor.AI.Examples/07_ParallelAgents/Program.cs) |
| `swarm` | [`17_swarm_orchestration.py`](https://github.com/conductor-oss/python-sdk/blob/main/examples/agents/17_swarm_orchestration.py) | [`Example17SwarmOrchestration.java`](https://github.com/conductor-oss/java-sdk/blob/main/agent-examples/src/main/java/org/conductoross/conductor/ai/examples/Example17SwarmOrchestration.java) | [`17-swarm-orchestration.ts`](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/agents/17-swarm-orchestration.ts) | [`17_SwarmOrchestration`](https://github.com/conductor-oss/csharp-sdk/blob/main/Conductor.AI.Examples/17_SwarmOrchestration/Program.cs) |
| `random` | [`16_random_strategy.py`](https://github.com/conductor-oss/python-sdk/blob/main/examples/agents/16_random_strategy.py) | [`Example16RandomStrategy.java`](https://github.com/conductor-oss/java-sdk/blob/main/agent-examples/src/main/java/org/conductoross/conductor/ai/examples/Example16RandomStrategy.java) | [`16-random-strategy.ts`](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/agents/16-random-strategy.ts) | [`16_RandomStrategy`](https://github.com/conductor-oss/csharp-sdk/blob/main/Conductor.AI.Examples/16_RandomStrategy/Program.cs) |
| `manual` | [`18_manual_selection.py`](https://github.com/conductor-oss/python-sdk/blob/main/examples/agents/18_manual_selection.py) | [`Example18ManualSelection.java`](https://github.com/conductor-oss/java-sdk/blob/main/agent-examples/src/main/java/org/conductoross/conductor/ai/examples/Example18ManualSelection.java) | [`18-manual-selection.ts`](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/agents/18-manual-selection.ts) | [`18_ManualSelection`](https://github.com/conductor-oss/csharp-sdk/blob/main/Conductor.AI.Examples/18_ManualSelection/Program.cs) |
| `plan_execute` | [`108_plan_execute_refs.py`](https://github.com/conductor-oss/python-sdk/blob/main/examples/agents/108_plan_execute_refs.py) | [`Example108PlanExecuteRefs.java`](https://github.com/conductor-oss/java-sdk/blob/main/agent-examples/src/main/java/org/conductoross/conductor/ai/examples/Example108PlanExecuteRefs.java) | [`108-plan-execute-refs.ts`](https://github.com/conductor-oss/javascript-sdk/blob/main/examples/agents/108-plan-execute-refs.ts) | [`108_PlanExecuteRefs`](https://github.com/conductor-oss/csharp-sdk/blob/main/Conductor.AI.Examples/108_PlanExecuteRefs/Program.cs) |

`round_robin` 还没有专属示例；它与 `random` 形状相同，只是替换策略值。

## 后续步骤

- [多 agent 交接配方](cookbook/agent-handoff.md) — 一个带三个专家的可运行 supervisor
- [大规模并行 agent](cookbook/agent-scatter-gather.md) — 扇出到 100 个子 agent
- [Agent 配置](agent-configuration.md) — agent 上还能设置什么
