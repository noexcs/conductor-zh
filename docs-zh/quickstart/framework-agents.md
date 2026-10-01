---
description: 通过 Conductor 的持久化运行时运行现有的 OpenAI Agents、Google ADK、LangChain 或 LangGraph 代理。
---

# 带你的框架代理

**结果：** 你的框架代理通过 Conductor 运行，并生成一个可检查的执行（实例）。

本页面向你已在其他框架（例如 OpenAI Agents、LangChain、LangGraph 或 Google ADK）中构建的代理。你保留框架已定义的代理对象，Conductor SDK 将其编译并运行为一个持久化、可检查的 Conductor 执行。如果你是从头开始，请用[你的第一个代理](first-agent.md)构建一个原生代理。

<section class="framework-hero" aria-labelledby="framework-quickstarts-title">
  <h2 id="framework-quickstarts-title">带上你现有的代理。</h2>
  <div class="framework-logo-grid framework-logo-grid--quickstart">
    <a class="framework-logo-card" href="#openai-agents-sdk" aria-label="OpenAI Agents SDK quickstart">
      <img class="framework-logo framework-logo--wide" src="../assets/images/frameworks/openai.svg" alt="" />
      <span>OpenAI Agents</span>
    </a>
    <a class="framework-logo-card" href="#langchain" aria-label="LangChain quickstart">
      <img class="framework-logo" src="../assets/images/frameworks/langchain.svg" alt="" />
      <span>LangChain</span>
    </a>
    <a class="framework-logo-card" href="#google-adk" aria-label="Google ADK quickstart">
      <img class="framework-logo" src="../assets/images/frameworks/google-adk.svg" alt="" />
      <span>Google ADK</span>
    </a>
    <a class="framework-logo-card" href="https://github.com/conductor-oss/javascript-sdk/tree/main/examples/agents/vercel-ai" aria-label="Vercel AI SDK examples on GitHub">
      <img class="framework-logo" src="../assets/images/frameworks/vercel.svg" alt="" />
      <span>Vercel AI SDK</span>
    </a>
  </div>
</section>

## 前提条件

首先，完成[连接到 Conductor](connect.md)，以便运行时能够访问你的服务器。然后确保服务器能够调用你的模型提供商。在 Developer Edition 上，将该提供商添加为[AI/LLM 集成](https://orkes.io/content/category/integrations/ai-llm)；在本地服务器上，启动之前[导出提供商 API 密钥](../devguide/ai/llm-orchestration.md#supported-llm-providers)。下面每个框架部分都以安装命令开始。大多数示例使用 OpenAI 模型，Google ADK 示例使用 Gemini，因此请提供相应的凭据。

## OpenAI Agents SDK

安装支持 OpenAI Agents 的 Conductor SDK：

```bash
pip install conductor-python
```

保存为 `openai_agent.py`：

```python
from conductor.ai import Runner
from agents import Agent, function_tool

@function_tool
def get_weather(city: str) -> str:
    return f"72F and sunny in {city}"

agent = Agent(
    name="weather_assistant",
    model="gpt-4o-mini",
    tools=[get_weather],
    instructions="You are a helpful assistant.",
)

result = Runner.run_sync(agent, "What's the weather in NYC?")
print(result.final_output)
```

运行 `python openai_agent.py`，然后在 UI 中验证输出和执行。唯一的 runner 导入发生变化：使用 `conductor.ai.Runner` 而不是框架的 runner。

## LangChain

安装支持 LangChain 的 Conductor SDK：

```bash
pip install 'conductor-python[langchain]'
```

```python
from conductor.ai.agents import AgentRuntime
from langchain.agents import create_agent
from langchain_core.tools import tool

@tool
def check_token() -> str:
    """Check a token."""
    return "available"

agent = create_agent("openai:gpt-4o-mini", tools=[check_token],
                     system_prompt="You are a helpful assistant.")

with AgentRuntime() as runtime:
    result = runtime.run(agent, "Is the token set?")
    result.print_result()
```

## LangGraph

安装支持 LangGraph 的 Conductor SDK：

```bash
pip install 'conductor-python[langgraph]'
```

```python
import math
from conductor.ai.agents import AgentRuntime
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent

@tool
def calculate(expression: str) -> str:
    """Evaluate a limited math expression."""
    return str(eval(expression, {"__builtins__": {}}, {"sqrt": math.sqrt, "pi": math.pi}))

graph = create_react_agent(
    ChatOpenAI(model="gpt-4o-mini", temperature=0), tools=[calculate], name="math_agent"
)

with AgentRuntime() as runtime:
    result = runtime.run(graph, "What is sqrt(256) + 2**10?")
    result.print_result()
```

## Google ADK

安装支持 Google ADK 的 Conductor SDK：

```bash
python -m pip install 'conductor-python[adk]'
```

```python
from conductor.ai.agents import AgentRuntime
from google.adk.agents import Agent

agent = Agent(
    name="adk_greeter",
    model="gemini-2.0-flash",
    instruction="You are friendly and concise.",
)

with AgentRuntime() as runtime:
    result = runtime.run(agent, "Say hello and share an ML fact.")
    result.print_result()
```

将该文件保存为 `adk_agent.py`，然后运行 `python adk_agent.py`。

## 验证与恢复

对于每个框架，验证打印的结果，并在 Conductor UI 中找到对应的执行。如果失败，首先检查运行时服务器 URL、框架包和提供商凭据；然后检查失败的任务，再重试。在确认某个代理动作的幂等性和恢复策略之前，不要重试可能已产生外部副作用的代理动作。

## 走向生产的下一步

**下一步：** [设计模式 → 代理配方](../devguide/ai/cookbook/index.md) 中的每一项都是完整的、可运行的示例 —— 交接、记忆、护栏、并行代理等。

使用[生产级代理架构](../devguide/ai/production-agent-architecture.md)添加治理、评估、部署、组合和运维。[Python SDK 框架代理指南](https://github.com/conductor-oss/python-sdk/blob/main/docs/agents/framework-agents.md)仍然是当前框架代理 API 和支持矩阵的权威来源。

## SDK 示例

使用受维护的 SDK 示例获取完整的、可运行的项目。破折号表示没有受维护示例的组合。

| 框架 | Python | Java | TypeScript / JavaScript | C# |
|---|---|---|---|---|
| OpenAI Agents | [示例](https://github.com/conductor-oss/python-sdk/tree/main/examples/agents/openai) | [示例](https://github.com/conductor-oss/java-sdk/tree/main/agent-examples) | [示例](https://github.com/conductor-oss/javascript-sdk/tree/main/examples/agents/openai) | [示例](https://github.com/conductor-oss/csharp-sdk/tree/main/Conductor.AI.Examples) |
| Google ADK | [示例](https://github.com/conductor-oss/python-sdk/tree/main/examples/agents/adk) | [示例](https://github.com/conductor-oss/java-sdk/tree/main/agent-examples) | [示例](https://github.com/conductor-oss/javascript-sdk/tree/main/examples/agents/adk) | [示例](https://github.com/conductor-oss/csharp-sdk/tree/main/Conductor.AI.Examples) |
| LangChain | [示例](https://github.com/conductor-oss/python-sdk/tree/main/examples/agents) | [LangChain4j 示例](https://github.com/conductor-oss/java-sdk/tree/main/agent-examples) | [示例](https://github.com/conductor-oss/javascript-sdk/tree/main/examples/agents) | — |
| LangGraph | [示例](https://github.com/conductor-oss/python-sdk/tree/main/examples/agents/langgraph) | [LangGraph4j 示例](https://github.com/conductor-oss/java-sdk/tree/main/agent-examples) | [示例](https://github.com/conductor-oss/javascript-sdk/tree/main/examples/agents/langgraph) | — |
| Vercel AI SDK | — | — | [示例](https://github.com/conductor-oss/javascript-sdk/tree/main/examples/agents/vercel-ai) | — |
