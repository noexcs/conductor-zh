---
description: 使用 Google ADK 编写一个非变更性的订单异常分诊智能体，通过 Conductor SDK 部署并调用它。
---

# ADK 分诊

```mermaid
flowchart LR
  G(["用 Google ADK 编写"]) --> B("通过<br/>Conductor SDK 部署")
  B --> A("像任何其他<br/>代理一样被调用")
  A --> O(["分诊建议"])
```

**结果：** 使用 Google ADK 编写一个非变更性的订单异常分诊代理，并通过 Conductor 调用它。

## 前置条件与编写路径

当前 Python SDK 快速入门使用 `python -m pip install 'conductor-python[adk]'`、`google.adk.agents.Agent` 和 `AgentRuntime`。在更改安装方式或框架代理调用之前，请先核对所属的 [Python SDK 框架指南](https://github.com/conductor-oss/python-sdk/blob/main/docs/agents/framework-agents.md)。

```python
from conductor.ai.agents import AgentRuntime
from google.adk.agents import Agent
from google.adk.tools.mcp_tool import McpToolset, StreamableHTTPConnectionParams

agent = Agent(
    name="adk_order_exception_triage",
    model="openai/gpt-4o",
    instruction="Use MCP evidence to recommend a disposition; never execute it.",
    tools=[McpToolset(connection_params=StreamableHTTPConnectionParams(url="http://127.0.0.1:3001/mcp"))],
)
with AgentRuntime() as runtime:
    runtime.run(agent, "Order O-42 arrived damaged.")
```

将配套的 [`deploy_local_cookbook_agents.py`](assets/deploy_local_cookbook_agents.py) 下载到工作目录；它会创建这个由 ADK 编写的能力。部署一次，并在调用父工作流之前保持工具工作者持续运行：

```bash
python3 deploy_local_cookbook_agents.py deploy
python3 deploy_local_cookbook_agents.py serve
```

输入是 `orderId` 和 `exception`；输出是一条建议。代理不得持有退款、履约或客户通知的凭证。

## 可运行的定义

将此保存为 `google-adk-order-triage.json`：

```json
--8<-- "docs/devguide/ai/cookbook/assets/google-adk-order-triage.json"
```

## 注册并运行

```bash
conductor workflow create google-adk-order-triage.json
conductor workflow start -w google_adk_order_exception_triage --sync -i '{"orderId":"O-42","exception":"Package damaged in transit."}'
```

## 生产环境注意事项

- **`agentType` 是 `conductor`，而不是 `adk`。** 由 Conductor SDK 运行它。
- **使用你的服务器实际配置过的模型** —— 如果已配置 Gemini，则使用 `gemini-2.0-flash`。
- **在部署中限制工具访问和迭代预算。**
- **它只建议处置方式，从不执行处置。** 将该操作路由到审批流程。
- **按订单 ID 加异常事件 ID 进行核对。**
