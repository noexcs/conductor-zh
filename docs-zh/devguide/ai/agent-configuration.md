---
description: Conductor agent 上的每一项设置，以及唯一重要的区分——部署时固定的内容，对比调用方可按次覆盖的内容。
---

# Agent 配置

agent 有两类设置，混淆它们是意外的常见来源。

- **定义设置**是 agent 的一部分。它们在 `deploy()` 时被编译进工作流，只有重新部署才会改变。
- **运行设置**由调用方在每次 `run()`、`start()` 或 `stream()` 时提供。它们从不改变已部署的 agent。

少数几项——模型、temperature 和 token 上限——可以在*两处*都设置。此时，**运行设置仅在本次执行中生效。**

## 定义设置

设置在 `Agent(...)` 上。直到下次 `deploy()` 前保持不变。

| 设置 | 默认值 | 作用 |
|---|---|---|
| `name` | *必填* | 调用方解析所用的名称。更改它会部署一个不同的 agent |
| `model` | `""` | 带提供商标识的模型，如 `openai/gpt-4o` |
| `instructions` | `""` | 系统提示词 |
| `tools` | `[]` | 模型可以调用的工具 |
| `guardrails` | `[]` | 对输入或输出的检查——参见 [Agent 护栏](agent-guardrails.md) |
| `agents` | `[]` | 多智能体系统的子 agent |
| `strategy` | `handoff` | 子 agent 的编排方式——参见 [多智能体架构](multi-agent-architecture.md) |
| `max_turns` | `25` | 模型轮次的硬上限。失控循环的主要控制手段 |
| `max_tokens` | `None` | 每次模型调用的上限 |
| `temperature` | `None` | 采样温度 |
| `context_window_budget` | `None` | 上下文压缩前的 token 预算 |
| `metadata` | `{}` | 随定义携带的任意标签 |

### 能力设置

它们决定 agent 能触及什么。有意设计为仅限定义——调用方不应能在运行时扩大它们。

| 设置 | 默认值 | 作用 |
|---|---|---|
| `cli_commands` | `False` | 挂载沙箱化的 `run_command` 工具 |
| `cli_allowed_commands` | `[]` | 命令白名单。其他一律拒绝 |
| `cli_config` | `None` | 完整 `CliConfig`——`timeout`、`working_dir`、`allow_shell` |
| `local_code_execution` | `False` | 允许 agent 执行代码 |
| `allowed_languages` | `[]` | 代码执行允许的语言 |
| `code_execution` | `None` | 完整的代码执行配置 |
| `credentials` | `[]` | 服务端在调用期间注入的密钥 |
| `prefill_tools` | `[]` | 第一轮之前预置的工具结果 |

## 运行设置

传给 `run()`、`start()` 或 `stream()`。只作用于一次执行。

| 设置 | 作用 |
|---|---|
| `prompt` | 本次运行的输入 |
| `version` | 固定某个已部署版本 |
| `media` | 本次运行的文件或图片 |
| `session_id` | 把多次运行串成一个会话 |
| `idempotency_key` | 让重试返回原运行，而不是启动新运行 |
| `timeout` | 本次执行的挂钟时间上限 |
| `context` | 运行可用的额外键值 |
| `credentials` | 本次执行的密钥 |
| `on_event` | 流式事件的回调 |
| `run_settings` | 按运行的模型覆盖——见下文 |

### 为单次运行覆盖模型

`RunSettings` 是不重新部署就更换模型选择的出口：

```python
from conductor.ai.agents import RunSettings

result = runtime.run(
    agent,
    "Summarise this incident.",
    run_settings=RunSettings(
        model="openai/gpt-4o",       # overrides the definition's model
        temperature=0.1,
        max_tokens=800,
        reasoning_effort="high",
        thinking_budget_tokens=2000,
    ),
)
```

`reasoning_effort` 和 `thinking_budget_tokens` 仅用于运行——没有对应的定义设置。

## 谁生效

| 设置 | 定义 | 运行 | 结果 |
|---|---|---|---|
| `model` | ✓ | ✓ (`RunSettings`) | 运行生效，仅限本次执行 |
| `temperature` | ✓ | ✓ (`RunSettings`) | 运行生效，仅限本次执行 |
| `max_tokens` | ✓ | ✓ (`RunSettings`) | 运行生效，仅限本次执行 |
| `credentials` | ✓ | ✓ | 运行向定义集合追加 |
| `max_turns`, `tools`, `guardrails`, `agents`, `strategy` | ✓ | — | 仅定义。更改需重新部署 |
| `reasoning_effort`, `thinking_budget_tokens` | — | ✓ | 仅运行 |
| `session_id`, `idempotency_key`, `media`, `context` | — | ✓ | 仅运行 |

## 生产注意事项

- **按设计，一切扩大触及范围的都是仅限定义。** 工具、护栏和 CLI 白名单不能被调用方放宽。
- **`max_turns` 是你的循环上界。** 默认 25 对简单 agent 已很宽裕；大批量运行的场景请调低。
- **任何会被重试的都用 `idempotency_key`。** 没有它，重试就是一次新的执行。
- **`session_id` 构成会话。** 没有它的运行彼此独立。
- **对关键调用方固定 `version`，** 让重新部署不会在它们脚下改变行为。
- **密钥放 `credentials`，绝不放 `instructions`。** 它们为调用注入，不会存储在定义中。

## 下一步

- [部署 Agent](deploying-agents.md)——定义设置何时真正生效
- [多智能体架构](multi-agent-architecture.md)——`strategy` 字段详解
- [Agent 护栏](agent-guardrails.md)——`guardrails` 字段详解
