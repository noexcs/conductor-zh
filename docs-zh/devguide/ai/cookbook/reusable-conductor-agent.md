# Conductor 代理

```mermaid
flowchart LR
  A(["你<br/>用 SDK 编写的代理"]) --> D("部署一次，<br/>保持运行")
  D --> P("任何工作流<br/>现在都可以调用它")
  P --> O(["持久化的代理运行"])
```

**结果：** 把一个用 SDK 编写的代理部署为稳定的能力，并从父工作流中调用它。

## 编写一个带护栏的 Conductor Agent（Python）

Python SDK 当前的护栏 API 使用 `RegexGuardrail`、`Position`、`OnFail` 和 `@tool`。这个入门示例会在一个本已获批的可写入工具运行之前，拦截信用卡形态的输入；`approval_required=True` 创建一个持久化的人的决策点。

```python
from conductor.ai.agents import Agent, AgentRuntime, OnFail, Position, RegexGuardrail, mcp_tool, tool

no_card_data = RegexGuardrail(
    patterns=[r"\b(?:\d[ -]?){15}\d\b"],
    name="no_card_data_in_email",
    position=Position.INPUT,
    on_fail=OnFail.RAISE,
    message="Refusing to send payment-card data by email.",
)

@tool(guardrails=[no_card_data], approval_required=True)
def notify_ops(summary: str) -> dict:
    # Call your idempotent, approved notification integration here.
    return {"status": "queued", "summary": summary}

agent = Agent(
    name="guarded-incident-planner",
    model="openai/gpt-4o",
    instructions="Summarize incidents and request approval before notification.",
    tools=[mcp_tool("http://127.0.0.1:3001/mcp"), notify_ops],
)

with AgentRuntime() as runtime:
    runtime.run(agent, "Summarize the incident and notify ops.").print_result()
```

为了可运行的本地部署，将配套的 [`deploy_local_cookbook_agents.py`](assets/deploy_local_cookbook_agents.py) 下载到工作目录。它把该能力部署为 `guarded-incident-planner`，并保持其工具工作者可用：

```bash
python3 deploy_local_cookbook_agents.py deploy
python3 deploy_local_cookbook_agents.py serve
```

父工作流锁定 `guarded-incident-planner`。策略模式参见 [代理护栏](../agent-guardrails.md)，并在推广之前测试护栏。

## 可运行的定义

将此保存为 `reusable-conductor-agent.json`：

```json
--8<-- "docs/devguide/ai/cookbook/assets/reusable-conductor-agent.json"
```

## 注册并运行

```bash
conductor workflow create reusable-conductor-agent.json
conductor workflow start -w invoke_reusable_conductor_agent --sync -i '{"prompt":"Summarize the incident evidence."}'
```

## 生产环境注意事项

- **在你的发布流程中锁定代理的名称和版本。** 父工作流按名称解析它。
- **用执行 ID 核对重试和取消。**
- **不要重试代理的副作用，** 除非其工具是幂等的。
- **大产出物以引用方式附加，** 而不是内联。
