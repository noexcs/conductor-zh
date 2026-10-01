---
description: 把 agent 放到 Conductor 服务端的四种方式——plan、run、deploy 和 serve——以及哪种属于 CI、哪种属于生产、哪种属于你的办公桌。
---

# 部署 Agent

你在 SDK 里写的 agent，在被放到服务端之前只是一个定义。`AgentRuntime` 为此提供了四个动词，它们之间的区别就是脚本与已部署能力之间的区别。

| 动词 | 作用 | 适用位置 |
|---|---|---|
| `plan()` | 把 agent 编译为工作流定义并返回。不注册任何东西，也不运行。 | 开发与 CI |
| `run()` | 需要时注册，执行一次，阻塞等待结果。 | 你的办公桌 |
| `deploy()` | 在服务端注册一个带名称、带版本的 agent。不执行。 | 发布流水线 |
| `serve()` | 启动执行 agent 工具的长生命周期工作者进程。 | 生产环境，作为服务 |

## plan——在任何东西运行之前看到图

`plan()` 编译 agent 并返回工作流定义。不写服务端，不执行。

```python
with AgentRuntime() as runtime:
    definition = runtime.plan(agent)
```

这是最便宜的检查，也是最多人跳过的。在 CI 中对输出做 diff，评审者就能看到某人编辑指令或添加工具时图中到底变了什么。

## run——一次执行，阻塞

```python
with AgentRuntime() as runtime:
    result = runtime.run(agent, "What's the weather in San Francisco?")
    result.print_result()
    print(result.execution_id)
```

`run()` 是开发循环：agent 不存在就注册，执行一次，阻塞到出结果。它还会在进程内启动所需的工作者，所以带工具的脚本无需你额外运行任何东西就能工作。

这种进程内的便利恰恰说明它不是生产模式——脚本退出，工作者也随之消失。

还有两个非阻塞的姊妹方法：

- **`start()`** 立即返回一个 `AgentHandle`，而不是等待。
- **`stream()`** 返回一个 `AgentStream`，让你可以在事件发生时消费它们。

## deploy——注册一个带名称、带版本的能力

```python
with AgentRuntime() as runtime:
    runtime.deploy(agent)
```

`deploy()` 之后，agent 以它的名称存在于服务端，任何东西都可以调用它——工作流中的 `AGENT` 任务、API、调度——无需你的代码参与。这正是 agent 成为共享能力、而不是某人运行的脚本的原因。

`deploy()` 可以一次接受多个 agent，并支持 `packages=`：

```python
runtime.deploy(billing_agent, support_agent)
```

部署后，用 CLI 或 API（而不是代码）给 agent 排上日程——参见[调度 Agent](scheduling-agents.md)。

## serve——执行实际工作的工作者进程

部署注册的是定义。它不会启动任何能执行你的 Python 工具的东西。`serve()` 就是那个进程：

```python
with AgentRuntime() as runtime:
    runtime.serve(agent)          # blocks
    # runtime.serve(agent, blocking=False)   # returns, for tests
```

如果已部署 agent 的执行停在 scheduled 状态一直不推进，几乎总是这个原因：没有人在 serve 它的工作者。

## 生产形态

把两者分开，在不同时间运行：

```python
# release.py — runs once in CI/CD
with AgentRuntime() as runtime:
    runtime.deploy(agent)

# worker.py — runs continuously as a service
with AgentRuntime() as runtime:
    runtime.serve(agent)
```

调用方随后按名称调用 agent，从不导入你的代码：

```json
{
  "name": "run_agent",
  "taskReferenceName": "run_agent_ref",
  "type": "AGENT",
  "inputParameters": { "agentType": "conductor", "name": "your-agent-name" }
}
```

## 控制运行中的执行

一旦有东西在运行，`AgentRuntime` 也是控制平面。每个方法都接受一个 `execution_id`：

| 方法 | 用途 |
|---|---|
| `get_status(id)` | 它在哪里 |
| `pause(id)` / `resume(id, agent)` | 挂起与继续 |
| `cancel(id, reason)` / `stop(id)` | 结束它 |
| `approve(id)` / `reject(id, reason)` | 回应一个人工门禁 |
| `respond(id, output)` / `send_message(id, msg)` / `signal(id, msg)` | 喂入一些东西 |

## 生产注意事项

- **`deploy()` 和 `serve()` 是不同的工作。** 从笔记本部署却从不 serve，是得到卡住执行的最常见方式。
- **有意地做版本管理。** 调用方按名称解析；变更不向后兼容时，在调用方固定版本。
- **`plan()` 属于 CI。** 它是唯一不碰服务端就能评审图变更的方式。
- **脚本里的 `run()` 不是部署。** 工作者随进程消亡。
- **在工具能运行的地方 serve。** CLI 工具、文件访问和凭证都在工作者进程中解析，而不是在服务端。

## 下一步

- [Agent 配置](agent-configuration.md)——哪些在部署时固定、哪些可以按运行覆盖
- [调度 Agent](scheduling-agents.md)——部署时挂一个 cron 调度
- [Conductor agent 配方](cookbook/reusable-conductor-agent.md)——从工作流调用的已部署 agent
