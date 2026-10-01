---
description: 运行你的第一个 Conductor Agent，并使用你选择的 SDK 语言验证其持久化执行。
---

# 你的第一个代理

**结果：** 一次已完成的代理运行，它被编译为 Conductor 工作流并执行。

**用时：** 大约 5 分钟。

Conductor Agents 支持 Python、Java、TypeScript/JavaScript 和 C#。在下方选择一种语言，查看其完整的安装和首次运行步骤。

若要使用已有的框架代理（例如 LangChain），请使用[框架代理快速入门](framework-agents.md)。你也可以使用 Conductor 的[原生 AI 任务](../devguide/ai/llm-orchestration.md)直接在工作流任务中调用 LLM 和工具。

## 前提条件

完成[连接到 Conductor](connect.md)，包括所选模型所需的托管模型集成或本地提供商 API 密钥配置。你还需要所选语言的运行时或 SDK 工具。

## 特定语言的快速入门

`CONDUCTOR_SERVER_URL` 连接变量（以及需要时的 `CONDUCTOR_AUTH_KEY`/`CONDUCTOR_AUTH_SECRET`）已在[连接到 Conductor](connect.md)中配置。将提供商凭据保存在代理工作者使用的环境或密钥系统中；不要将它们放入工作流输入中。

<div class="agent-language-picker" markdown="1">
  <label for="agent-language-select">语言</label>
  <select id="agent-language-select" aria-describedby="agent-language-help">
    <option value="python" selected>Python</option>
    <option value="java">Java</option>
    <option value="typescript">TypeScript / JavaScript</option>
    <option value="csharp">C#</option>
  </select>
  <p id="agent-language-help">选择一种语言，以显示其安装和可运行的第一个代理步骤。</p>

  <section class="agent-language-guide" data-agent-language="python" markdown="1">

<p class="agent-language-guide__heading" role="heading" aria-level="3">1. 安装 Python 支持</p>

```bash
pip install conductor-python
```

<p class="agent-language-guide__heading" role="heading" aria-level="3">2. 保存并运行代理</p>

将其保存为 `hello.py`：

```python
from conductor.ai.agents import Agent, AgentRuntime

agent = Agent(
    name="greeter",
    model="openai/gpt-4o-mini",
    instructions="You are a friendly assistant. Keep responses brief.",
)

with AgentRuntime() as runtime:
    result = runtime.run(agent, "Say hello and share a fun Python fact.")
    result.print_result()
```

```bash
python hello.py
```

更多示例，参见 [Python 代理指南](https://github.com/conductor-oss/python-sdk/tree/main/docs/agents)。

  </section>

  <section class="agent-language-guide" data-agent-language="java" hidden markdown="1">

<p class="agent-language-guide__heading" role="heading" aria-level="3">1. 安装 Java 支持</p>

Gradle：

```groovy
dependencies {
    implementation 'org.conductoross:conductor-client-ai:VERSION'
}
```

Maven：

```xml
<dependency>
    <groupId>org.conductoross</groupId>
    <artifactId>conductor-client-ai</artifactId>
    <version>VERSION</version>
</dependency>
```

<p class="agent-language-guide__heading" role="heading" aria-level="3">2. 定义并运行代理</p>

```java
import org.conductoross.conductor.ai.Agent;
import org.conductoross.conductor.ai.AgentRuntime;
import org.conductoross.conductor.ai.model.AgentResult;

Agent agent = Agent.builder()
    .name("java_greeter")
    .model("openai/gpt-4o-mini")
    .instructions("You are friendly and concise.")
    .build();

try (AgentRuntime runtime = new AgentRuntime()) {
    AgentResult result = runtime.run(agent, "Share a fun Java fact.");
    result.printResult();
}
```

使用你的 Gradle 或 Maven 应用任务运行该类。项目设置和可运行示例，参见 [Java 代理指南](https://github.com/conductor-oss/java-sdk/tree/main/docs/agents)。

  </section>

  <section class="agent-language-guide" data-agent-language="typescript" hidden markdown="1">

<p class="agent-language-guide__heading" role="heading" aria-level="3">1. 安装 TypeScript / JavaScript 支持</p>

```bash
npm install @io-orkes/conductor-javascript
```

<p class="agent-language-guide__heading" role="heading" aria-level="3">2. 保存并运行代理</p>

将其保存为 `my-agent.ts`：

```typescript
import { Agent, AgentRuntime } from "@io-orkes/conductor-javascript/agents";

const agent = new Agent({
  name: "greeter",
  model: "openai/gpt-4o-mini",
  instructions: "You are friendly and concise.",
});

const runtime = new AgentRuntime();
try {
  const result = await runtime.run(agent, "Share a fun TypeScript fact.");
  result.printResult();
} finally {
  await runtime.shutdown();
}
```

```bash
npx tsx my-agent.ts
```

更多示例，参见 [TypeScript 代理指南](https://github.com/conductor-oss/javascript-sdk/tree/main/docs/agents)。

  </section>

  <section class="agent-language-guide" data-agent-language="csharp" hidden markdown="1">

<p class="agent-language-guide__heading" role="heading" aria-level="3">1. 安装 C# 支持</p>

```bash
dotnet add package conductor-ai
```

<p class="agent-language-guide__heading" role="heading" aria-level="3">2. 定义并运行代理</p>

```csharp
using Conductor.AI;

var agent = new Agent("greeter")
{
    Model = "openai/gpt-4o-mini",
    Instructions = "You are friendly and concise.",
};

await using var runtime = new AgentRuntime();
var result = await runtime.RunAsync(agent, "Share a fun C# fact.");
result.PrintResult();
```

```bash
dotnet run
```

项目设置和可运行示例，参见 [C# 代理指南](https://github.com/conductor-oss/csharp-sdk/tree/main/docs/agents)。

  </section>
</div>

<script>
  (function () {
    var select = document.getElementById("agent-language-select");
    var guides = document.querySelectorAll("[data-agent-language]");

    function showGuide() {
      guides.forEach(function (guide) {
        guide.hidden = guide.dataset.agentLanguage !== select.value;
      });
    }

    select.addEventListener("change", showGuide);
  })();
</script>

## 3. 验证与恢复

在 Conductor UI 中，找到由该运行创建的执行。验证其终止状态，并检查其任务时间线、输入和输出。如果运行无法访问模型，首先确认工作者环境中的服务器 URL 和提供商凭据；然后在执行中检查失败的任务，再重试。

## 将你的代理加入工作流

部署代理后，工作流可以将其作为一个 `AGENT` 任务调用，与普通 API 调用、检索、审批、重试、分支和并行工作并存。工作流拥有持久化的业务流程；代理拥有其中的模型驱动的决策或动作。

```json
{
  "name": "ask_agent",
  "taskReferenceName": "ask_agent_ref",
  "type": "AGENT",
  "inputParameters": {
    "agentType": "conductor",
    "name": "greeter",
    "prompt": "Summarize this workflow context: ${fetch_context.output.response.body}",
    "pollIntervalSeconds": 5
  }
}
```

该任务会记录代理执行 ID、状态、文本和结构化输出，因此运营人员可以一起检查父工作流和代理运行。参见[完整的工作流+代理示例](../devguide/ai/first-ai-agent.md)或 [`AGENT` 任务集成指南](../devguide/ai/conductor-agents.md#use-a-deployed-agent-in-a-workflow)。

## 你构建了什么

每种语言使用相同的持久化执行模型：运行时将代理编译为 Conductor 工作流并运行，保留可检查的执行记录。后续设计可以添加审批、等待、重试、组合和运维恢复，而无需将代理逻辑移入一个长驻进程。

## 走向生产的下一步

**下一步：** [带你的框架代理](framework-agents.md) —— 通过相同的持久化运行时运行现有的 OpenAI Agents、LangChain、LangGraph 或 ADK 代理。

继续学习[生产级代理架构](../devguide/ai/production-agent-architecture.md)。它涵盖治理、评估、部署、组合、恢复和运维。
