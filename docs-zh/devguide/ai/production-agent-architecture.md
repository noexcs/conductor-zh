---
description: "一个紧凑的、框架无关的蓝图：在 Conductor 上落地生产级 AI agent — 选择执行边界、定义契约、治理副作用，并运营持久化工作流。"
---

# 生产级 Agent 架构

本页是运行生产级 agent 的参考架构。核心思想：父工作流拥有业务流程，agent 在它内部一个显式的执行边界后面运行。仅凭 agent 一句话，不会发生任何不可逆的事情，因为父工作流在任何写入之前都会校验结果并施加审批。该模式与框架无关：边界后面的 agent 可以由原生任务构建、作为 Conductor Agent 部署，或通过 A2A 远程访问。

要查看该模式的可运行实现，参见[持久化自适应图](dynamic-workflows.md)：它构建了一个受治理的 PR 审查 agent，带受限的扇出，并在其唯一的副作用之前进行人工审批。

## 父工作流的参考路径

每条路径都始于父工作流，也终于父工作流：校验请求、选择一个执行边界、校验返回的结果，然后施加审批、写入或补偿。父工作流拥有业务流程；每条 agent 路径只拥有其边界后面的工作。

<div style="margin: 2rem 0;">
<svg viewBox="0 0 960 430" xmlns="http://www.w3.org/2000/svg" style="max-width: 960px; width: 100%; height: auto;" role="img" aria-labelledby="production-agent-parent-title production-agent-parent-desc">
  <title id="production-agent-parent-title">生产级 agent 父工作流</title>
  <desc id="production-agent-parent-desc">父工作流校验请求，选择原生任务、已部署的 Conductor Agent 或远程 A2A agent，校验返回的结果，然后在写入之前获取审批，或在失败时进行补偿。原生路径和已部署 agent 路径可在 Conductor 中观测；A2A 交接在父边界可观测，而其内部保持远程。</desc>
  <defs>
    <marker id="parent-arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M 0 0 L 10 5 L 0 10 z" fill="#4a5568"/></marker>
  </defs>

  <rect x="20" y="18" width="920" height="390" rx="12" fill="rgba(59,130,246,0.04)" stroke="#94a3b8" stroke-width="1.5"/>
  <text x="42" y="45" font-size="13" font-weight="600" fill="#2e3545" font-family="sans-serif">父工作流 — 持久化业务流程边界</text>

  <rect x="52" y="88" width="150" height="58" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/>
  <text x="127" y="113" text-anchor="middle" font-size="12" font-weight="600" fill="#2e3545" font-family="sans-serif">校验请求</text>
  <text x="127" y="130" text-anchor="middle" font-size="10" fill="#4a5568" font-family="sans-serif">形状、策略、ID</text>

  <line x1="202" y1="117" x2="256" y2="117" stroke="#4a5568" stroke-width="1.5" marker-end="url(#parent-arrow)"/>
  <polygon points="285,82 326,117 285,152 244,117" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/>
  <text x="285" y="113" text-anchor="middle" font-size="10" font-weight="600" fill="#2e3545" font-family="sans-serif">选择</text>
  <text x="285" y="127" text-anchor="middle" font-size="10" fill="#4a5568" font-family="sans-serif">执行边界</text>

  <line x1="326" y1="117" x2="372" y2="117" stroke="#4a5568" stroke-width="1.5" marker-end="url(#parent-arrow)"/>
  <line x1="285" y1="152" x2="285" y2="290" stroke="#4a5568" stroke-width="1.5"/>
  <line x1="285" y1="290" x2="372" y2="290" stroke="#4a5568" stroke-width="1.5" marker-end="url(#parent-arrow)"/>
  <line x1="326" y1="117" x2="372" y2="214" stroke="#4a5568" stroke-width="1.5" marker-end="url(#parent-arrow)"/>

  <rect x="372" y="72" width="220" height="90" rx="7" fill="rgba(6,214,160,0.10)" stroke="#06a88a" stroke-width="1.5"/>
  <text x="482" y="99" text-anchor="middle" font-size="12" font-weight="600" fill="#155e75" font-family="sans-serif">原生任务</text>
  <text x="482" y="118" text-anchor="middle" font-size="10" fill="#2e3545" font-family="sans-serif">LLM、MCP、控制流</text>
  <text x="482" y="140" text-anchor="middle" font-size="10" fill="#155e75" font-family="sans-serif">执行 + 可观测性: Conductor</text>

  <rect x="372" y="170" width="220" height="90" rx="7" fill="rgba(59,130,246,0.10)" stroke="#2563eb" stroke-width="1.5"/>
  <text x="482" y="197" text-anchor="middle" font-size="12" font-weight="600" fill="#1d4ed8" font-family="sans-serif">Conductor Agent</text>
  <text x="482" y="216" text-anchor="middle" font-size="10" fill="#2e3545" font-family="sans-serif">AGENT: agentType conductor</text>
  <text x="482" y="238" text-anchor="middle" font-size="10" fill="#1d4ed8" font-family="sans-serif">执行 + 可观测性: Conductor</text>

  <rect x="372" y="268" width="220" height="90" rx="7" fill="rgba(245,158,11,0.12)" stroke="#d97706" stroke-width="1.5"/>
  <text x="482" y="295" text-anchor="middle" font-size="12" font-weight="600" fill="#92400e" font-family="sans-serif">远程 A2A agent</text>
  <text x="482" y="314" text-anchor="middle" font-size="10" fill="#2e3545" font-family="sans-serif">AGENT: agentType a2a</text>
  <text x="482" y="336" text-anchor="middle" font-size="10" fill="#92400e" font-family="sans-serif">交接可观测；内部为远程</text>

  <line x1="592" y1="117" x2="656" y2="117" stroke="#4a5568" stroke-width="1.5"/>
  <line x1="592" y1="215" x2="656" y2="215" stroke="#4a5568" stroke-width="1.5"/>
  <line x1="592" y1="313" x2="656" y2="313" stroke="#4a5568" stroke-width="1.5"/>
  <line x1="656" y1="117" x2="656" y2="313" stroke="#4a5568" stroke-width="1.5"/>
  <line x1="656" y1="215" x2="708" y2="215" stroke="#4a5568" stroke-width="1.5" marker-end="url(#parent-arrow)"/>

  <rect x="708" y="186" width="178" height="58" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/>
  <text x="797" y="211" text-anchor="middle" font-size="12" font-weight="600" fill="#2e3545" font-family="sans-serif">校验结果</text>
  <text x="797" y="228" text-anchor="middle" font-size="10" fill="#4a5568" font-family="sans-serif">schema、策略、工件</text>

  <line x1="797" y1="244" x2="797" y2="278" stroke="#4a5568" stroke-width="1.5" marker-end="url(#parent-arrow)"/>
  <rect x="708" y="278" width="178" height="58" rx="6" fill="#f59e0b" stroke="#d97706" stroke-width="1.5"/>
  <text x="797" y="302" text-anchor="middle" font-size="12" font-weight="600" fill="#fff" font-family="sans-serif">审批后写入</text>
  <text x="797" y="319" text-anchor="middle" font-size="10" fill="rgba(255,255,255,0.85)" font-family="sans-serif">或失败时补偿</text>
</svg>
</div>

## 选择执行边界

父工作流可以使用这些执行路径中的一个或多个。根据 agent 行为应归属的位置选择路径；三条路径都参与同一个持久化业务流程。

- **原生 AI 任务**直接运行在工作流图中。当工作流定义就是 agent 实现时，使用 `LLM_CHAT_COMPLETE`、MCP 任务、`HUMAN` 和控制流任务。
- **已部署的 Conductor Agents** 通过 `agentType: "conductor"` 的 `AGENT` 任务运行。它们包括用 Conductor SDK 编写的 agent，或从 OpenAI Agents、Google ADK、LangChain、LangGraph 和 Vercel AI SDK 带来的 agent。Conductor 把这些 agent 编译为已部署的工作流图。
- **远程 A2A agent** 通过 `agentType: "a2a"` 的 `AGENT` 任务运行。这是对独立部署的 Agent2Agent 服务的一次持久化交接：Conductor 管理父工作流的生命周期，而远程服务保留自己的实现和内部细节。

`agentType` 选择的是执行模式，而不是指定编写框架。使用 `SUB_WORKFLOW` 或 `START_WORKFLOW` 组合子工作流；当父工作流调用一个 agent 运行时，使用 `AGENT`。

| 边界 | 何时使用 | 执行与可观测性 |
|---|---|---|
| 原生任务 | 工作流图拥有编排和 agent 行为。 | 原生系统任务执行，并可在 Conductor 中观测。 |
| `AGENT` / `agentType: "conductor"` | agent 用 Conductor SDK 编写，或来自受支持的框架：OpenAI Agents、Google ADK、LangChain、LangGraph 或 Vercel AI SDK。 | Conductor 编译并运行已部署的 agent 图，因此其执行可在 Conductor 中观测。 |
| `AGENT` / `agentType: "a2a"` | 专家独立部署为远程 A2A 服务。 | Conductor 观测持久化交接、生命周期和返回的工件；远程 agent 拥有自己的私有内部。 |
| `SUB_WORKFLOW` / `START_WORKFLOW` | 你在组合另一个 Conductor 工作流，同步或发后即忘（fire-and-forget）。 | 它们组合的是工作流定义；它们不调用任何一种 `AGENT` 运行时模式。 |

## 每个 agent 边界处的生产级契约

| 决策 | 默认生产级契约 |
|---|---|
| 输入与输出 | 在边界之前定义并校验输入，在边界之后校验输出；不要让未经校验的模型或远程响应来决定一个有重大影响的动作。 |
| 身份与副作用 | 把关联 ID 和幂等键带入外部副作用和远程交接。把每个工具和远程 agent 的副作用视为至少一次；使用幂等或显式的对账标记。 |
| 状态所有者 | 把编排状态放在工作流变量中，可恢复的已部署 agent 状态放在其执行 ID 之后，远程续接状态放在 A2A 上下文和任务 ID 中。 |
| 持久化负载 | 返回小的持久化工件和引用，而不是原始历史或大负载。 |

## 生产就绪

- 在服务器端解析凭据。永远不要把机密放进 prompt 或工作流输入。
- 使用最小权限工具，校验输出，并在有重大影响的写入之前要求人工审批。
- 为轮次、并行度、时间、token 或成本、重试、取消和补偿行为设置上限。
- 指定一个负责人，并定义一套关联 ID 约定。监控终止状态、耗时、重试、超时或取消、工具失败、预算耗尽和审批等待时长。
- 做一次恢复演练：中断一个安全的执行，通过关联 ID 定位它，视情况重试、恢复或终止，并校验审计轨迹。
- 让发布保持 KISS：在沙箱工具上测试变更的路径，部署它，并保留一个已知良好的定义用于回滚。

实现细节参见 [Conductor Agents](conductor-agents.md)、[框架 Agents](agent-framework-recipes.md)、[A2A 集成](a2a-integration.md)、[护栏](agent-guardrails.md)、[评估](agent-evals.md)、[失败语义](failure-semantics.md) 和 [持久化自适应图](dynamic-workflows.md)。

## 原生任务实现：架构图

<div style="margin: 2rem 0;">
<svg viewBox="0 0 720 820" xmlns="http://www.w3.org/2000/svg" style="max-width: 720px; width: 100%; height: auto;">
  <defs>
    <marker id="pa-arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M 0 0 L 10 5 L 0 10 z" fill="#4a5568"/></marker>
    <marker id="pa-arrow-teal" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M 0 0 L 10 5 L 0 10 z" fill="#06d6a0"/></marker>
    <marker id="pa-arrow-red" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M 0 0 L 10 5 L 0 10 z" fill="#dc2626"/></marker>
    <marker id="pa-arrow-blue" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M 0 0 L 10 5 L 0 10 z" fill="#3b82f6"/></marker>
  </defs>

  <!-- Background for loop region -->
  <rect x="30" y="215" width="660" height="480" rx="12" fill="rgba(6,214,160,0.06)" stroke="#06d6a0" stroke-width="1.5" stroke-dasharray="6,4"/>
  <text x="50" y="240" font-size="11" font-weight="600" fill="#06d6a0" font-family="sans-serif">DO_WHILE — Agent 循环（每次迭代做检查点）</text>

  <!-- Start -->
  <circle cx="360" cy="30" r="22" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/>
  <text x="360" y="35" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif">开始</text>

  <!-- Discover Tools -->
  <line x1="360" y1="52" x2="360" y2="80" stroke="#4a5568" stroke-width="1.5" marker-end="url(#pa-arrow)"/>
  <rect x="245" y="80" width="230" height="40" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/>
  <text x="360" y="98" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif" font-weight="600">发现工具</text>
  <text x="360" y="112" text-anchor="middle" font-size="9" fill="#4a5568" font-family="sans-serif">LIST_MCP_TOOLS</text>

  <!-- Init Memory -->
  <line x1="360" y1="120" x2="360" y2="148" stroke="#4a5568" stroke-width="1.5" marker-end="url(#pa-arrow)"/>
  <rect x="245" y="148" width="230" height="40" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/>
  <text x="360" y="166" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif" font-weight="600">初始化记忆</text>
  <text x="360" y="180" text-anchor="middle" font-size="9" fill="#4a5568" font-family="sans-serif">SET_VARIABLE</text>

  <!-- Arrow into loop -->
  <line x1="360" y1="188" x2="360" y2="260" stroke="#4a5568" stroke-width="1.5" marker-end="url(#pa-arrow)"/>

  <!-- Plan (LLM) -->
  <rect x="245" y="260" width="230" height="45" rx="6" fill="#3b82f6" stroke="#2563eb" stroke-width="1.5"/>
  <text x="360" y="280" text-anchor="middle" font-size="11" fill="#fff" font-weight="600" font-family="sans-serif">规划下一步动作</text>
  <text x="360" y="296" text-anchor="middle" font-size="9" fill="rgba(255,255,255,0.8)" font-family="sans-serif">LLM_CHAT_COMPLETE</text>

  <!-- Switch: done / needs_approval / execute -->
  <line x1="360" y1="305" x2="360" y2="335" stroke="#4a5568" stroke-width="1.5" marker-end="url(#pa-arrow)"/>
  <polygon points="360,335 400,365 360,395 320,365" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/>
  <text x="360" y="362" text-anchor="middle" font-size="9" fill="#2e3545" font-family="sans-serif" font-weight="600">SWITCH</text>
  <text x="360" y="374" text-anchor="middle" font-size="8" fill="#4a5568" font-family="sans-serif">done?</text>

  <!-- Done branch (exits loop) — goes right and down to end -->
  <line x1="400" y1="365" x2="620" y2="365" stroke="#06d6a0" stroke-width="1.5"/>
  <text x="500" y="358" text-anchor="middle" font-size="9" fill="#06d6a0" font-family="sans-serif" font-weight="600">done = true</text>
  <line x1="620" y1="365" x2="620" y2="770" stroke="#06d6a0" stroke-width="1.5" marker-end="url(#pa-arrow-teal)"/>

  <!-- Needs approval branch — goes left -->
  <line x1="320" y1="365" x2="140" y2="365" stroke="#4a5568" stroke-width="1.5"/>
  <text x="230" y="358" text-anchor="middle" font-size="9" fill="#4a5568" font-family="sans-serif">needs_approval</text>
  <line x1="140" y1="365" x2="140" y2="420" stroke="#4a5568" stroke-width="1.5" marker-end="url(#pa-arrow)"/>

  <!-- Human Approval -->
  <rect x="55" y="420" width="170" height="45" rx="6" fill="#f59e0b" stroke="#d97706" stroke-width="1.5"/>
  <text x="140" y="440" text-anchor="middle" font-size="11" fill="#fff" font-weight="600" font-family="sans-serif">人工审批</text>
  <text x="140" y="456" text-anchor="middle" font-size="9" fill="rgba(255,255,255,0.8)" font-family="sans-serif">HUMAN（持久化暂停）</text>

  <!-- Arrow from approval to tool -->
  <line x1="140" y1="465" x2="140" y2="500" stroke="#4a5568" stroke-width="1.5"/>
  <line x1="140" y1="500" x2="360" y2="500" stroke="#4a5568" stroke-width="1.5"/>

  <!-- Execute branch — goes straight down -->
  <line x1="360" y1="395" x2="360" y2="490" stroke="#4a5568" stroke-width="1.5" marker-end="url(#pa-arrow)"/>
  <text x="375" y="440" font-size="9" fill="#4a5568" font-family="sans-serif">execute</text>

  <!-- Execute Tool -->
  <rect x="265" y="490" width="190" height="45" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/>
  <text x="360" y="510" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif" font-weight="600">执行工具</text>
  <text x="360" y="526" text-anchor="middle" font-size="9" fill="#4a5568" font-family="sans-serif">CALL_MCP_TOOL</text>

  <!-- Retry badge on tool -->
  <circle cx="465" cy="500" r="12" fill="#f59e0b" stroke="#d97706" stroke-width="1"/>
  <text x="465" y="504" text-anchor="middle" font-size="9" fill="#fff" font-weight="bold" font-family="sans-serif">!</text>
  <text x="485" y="504" font-size="8" fill="#4a5568" font-family="sans-serif">自动重试</text>

  <!-- Update Memory -->
  <line x1="360" y1="535" x2="360" y2="570" stroke="#4a5568" stroke-width="1.5" marker-end="url(#pa-arrow)"/>
  <rect x="255" y="570" width="210" height="40" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/>
  <text x="360" y="588" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif" font-weight="600">更新记忆</text>
  <text x="360" y="602" text-anchor="middle" font-size="9" fill="#4a5568" font-family="sans-serif">SET_VARIABLE</text>

  <!-- Budget check -->
  <line x1="360" y1="610" x2="360" y2="640" stroke="#4a5568" stroke-width="1.5" marker-end="url(#pa-arrow)"/>
  <polygon points="360,640 400,665 360,690 320,665" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/>
  <text x="360" y="662" text-anchor="middle" font-size="8" fill="#2e3545" font-family="sans-serif" font-weight="600">预算</text>
  <text x="360" y="674" text-anchor="middle" font-size="8" fill="#4a5568" font-family="sans-serif">检查</text>

  <!-- Loop back arrow -->
  <line x1="320" y1="665" x2="80" y2="665" stroke="#06d6a0" stroke-width="1.5"/>
  <line x1="80" y1="665" x2="80" y2="282" stroke="#06d6a0" stroke-width="1.5"/>
  <line x1="80" y1="282" x2="245" y2="282" stroke="#06d6a0" stroke-width="1.5" marker-end="url(#pa-arrow-teal)"/>
  <text x="68" y="480" text-anchor="middle" font-size="9" fill="#06d6a0" font-family="sans-serif" font-weight="600" transform="rotate(-90 68 480)">下一次迭代</text>

  <!-- Budget exceeded — exit loop -->
  <line x1="400" y1="665" x2="620" y2="665" stroke="#dc2626" stroke-width="1.5"/>
  <text x="510" y="658" text-anchor="middle" font-size="9" fill="#dc2626" font-family="sans-serif" font-weight="600">预算超限</text>
  <line x1="620" y1="665" x2="620" y2="770" stroke="#dc2626" stroke-width="1.5"/>

  <!-- End -->
  <line x1="360" y1="695" x2="360" y2="770" stroke="#4a5568" stroke-width="1.5" marker-end="url(#pa-arrow)"/>
  <circle cx="360" cy="792" r="22" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/>
  <text x="360" y="797" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif">结束</text>

  <!-- Compensation annotation -->
  <rect x="490" y="748" width="180" height="42" rx="6" fill="#fff" stroke="#dc2626" stroke-width="1" stroke-dasharray="4,3"/>
  <text x="580" y="765" text-anchor="middle" font-size="9" fill="#dc2626" font-family="sans-serif" font-weight="600">失败时:</text>
  <text x="580" y="780" text-anchor="middle" font-size="9" fill="#dc2626" font-family="sans-serif">failureWorkflow 运行</text>
  <text x="580" y="790" text-anchor="middle" font-size="9" fill="#dc2626" font-family="sans-serif">补偿</text>
  <line x1="490" y1="770" x2="385" y2="785" stroke="#dc2626" stroke-width="1" stroke-dasharray="3,3"/>

  <!-- Persistence annotation -->
  <rect x="500" y="260" width="150" height="50" rx="6" fill="#fff" stroke="#3b82f6" stroke-width="1" stroke-dasharray="4,3"/>
  <text x="575" y="278" text-anchor="middle" font-size="9" fill="#3b82f6" font-family="sans-serif" font-weight="600">每个步骤都持久化</text>
  <text x="575" y="292" text-anchor="middle" font-size="9" fill="#3b82f6" font-family="sans-serif">Prompt、响应、</text>
  <text x="575" y="304" text-anchor="middle" font-size="9" fill="#3b82f6" font-family="sans-serif">token、耗时</text>
  <line x1="500" y1="285" x2="475" y2="282" stroke="#3b82f6" stroke-width="1" stroke-dasharray="3,3"/>
</svg>
</div>


## 标准 Agent 模式

生产级 agent 有这些关注点。每一个都映射到一个具体的 Conductor 原语：

| Agent 关注点 | Conductor 原语 | 如何工作 |
|---|---|---|
| **规划下一步动作** | `LLM_CHAT_COMPLETE` | LLM 接收目标 + 上下文 + 工具列表，返回结构化计划 |
| **在运行时选择一个已批准的工具** | `SWITCH` + 带防护的 `CALL_MCP_TOOL` | LLM 提议一条路由；图在执行之前重新校验能力选择。 |
| **执行工具** | `CALL_MCP_TOOL`、`HTTP` 或 `SIMPLE` 工作者 | 工具带着重试策略、超时和完整的 I/O 记录运行 |
| **带退避的重试** | 任务定义 `retryLogic` | `FIXED`、`EXPONENTIAL_BACKOFF` 或 `LINEAR_BACKOFF` — 无需代码 |
| **并行工具调用** | `FORK/JOIN` 或 `FORK_JOIN_DYNAMIC` | 并行扇出到一组有界的工具，然后汇合它们的结果 |
| **记忆 / 上下文交接** | `SET_VARIABLE` + 工作流变量 | 跨循环迭代累积结果；传给下一次 LLM 调用 |
| **人工审批关卡** | `HUMAN` 任务 | 持久化暂停。熬过重启和部署。在 API 信号时恢复。 |
| **长等待（小时/天）** | `WAIT` 任务 | 基于定时器的持久化暂停。熬过服务器重启。 |
| **从外部事件恢复** | `HUMAN` 任务 + webhook/API | 外部系统调用 Task Update API。工作流带着负载恢复。 |
| **反思 / 评估循环** | 带 LLM-as-judge 的 `DO_WHILE` | 第二个 LLM 评估输出质量；低于阈值时循环继续 |
| **预算 / 迭代上限** | `DO_WHILE` `loopCondition` | 循环条件中是 `iteration < maxIterations`，或 token/成本检查 |
| **终止条件** | `DO_WHILE` 退出 + `SWITCH` | LLM 设置 `done: true`，或评估者判定目标已达成 |
| **调用已部署的专家 agent** | `agentType: "conductor"` 的 `AGENT` | 按名称运行一个已部署的 Conductor Agent；其编译后的图在 Conductor 中可见。 |
| **交接给远程专家 agent** | `agentType: "a2a"` 的 `AGENT` | 调用远程 A2A 服务；Conductor 在父边界持久化交接、生命周期和返回的工件。 |
| **组合子工作流** | `SUB_WORKFLOW` 或 `START_WORKFLOW` | 父工作流等待时用 `SUB_WORKFLOW`；发后即忘的工作流组合用 `START_WORKFLOW`。 |
| **失败时补偿** | `failureWorkflow` | 撤销副作用：吊销 API 调用、发送通知、释放资源 |
| **审计轨迹** | 自动 | 每个任务的输入、输出、耗时、重试次数和 worker ID 都被持久化 |


## 原生任务实现：端到端工作流

原生任务路径的可运行事实来源是本仓库 AI 示例目录中的 `ai/examples/35-governed-adaptive-agent.json`。每个步骤都是原生系统任务或操作符 — 没有自定义代码或外部框架。下面的紧凑 JSON 是该路径的概念性基线；部署时请使用受治理的 PR 审查器，因为它添加了上述生产级护栏。

```json
{
  "name": "production_agent",
  "description": "Reference architecture: durable production agent",
  "version": 1,
  "schemaVersion": 2,
  "inputParameters": ["goal", "mcpServerUrl", "maxIterations"],
  "tasks": [
    {
      "name": "discover_tools",
      "taskReferenceName": "discover",
      "type": "LIST_MCP_TOOLS",
      "inputParameters": {
        "mcpServer": "${workflow.input.mcpServerUrl}"
      }
    },
    {
      "name": "initialize_memory",
      "taskReferenceName": "init_memory",
      "type": "SET_VARIABLE",
      "inputParameters": {
        "last_action": "",
        "last_result": "",
        "final_answer": ""
      }
    },
    {
      "name": "agent_loop",
      "taskReferenceName": "loop",
      "type": "DO_WHILE",
      "loopCondition": "$.plan['route'] != 'done' && $.loop['iteration'] < $.maxIterations",
      "inputParameters": {
        "maxIterations": "${workflow.input.maxIterations}"
      },
      "loopOver": [
        {
          "name": "plan_next_action",
          "taskReferenceName": "plan",
          "type": "LLM_CHAT_COMPLETE",
          "inputParameters": {
            "llmProvider": "anthropic",
            "model": "claude-sonnet-4-20250514",
            "messages": [
              {
                "role": "system",
                "message": "You are a production AI agent. Goal: ${workflow.input.goal}\n\nAvailable tools: ${discover.output.tools}\n\nMost recent action: ${workflow.variables.last_action}\nMost recent result: ${workflow.variables.last_result}\n\nRespond with JSON only. Use {\"route\": \"execute\", \"action\": \"tool_name\", \"arguments\": {}, \"reasoning\": \"why\"} for a safe tool call, {\"route\": \"needs_approval\", \"action\": \"tool_name\", \"arguments\": {}, \"reasoning\": \"why\"} for a reviewable tool call, or {\"route\": \"done\", \"answer\": \"final answer\"} when complete."
              }
            ],
            "temperature": 0.1,
            "maxTokens": 1000,
            "jsonOutput": true
          }
        },
        {
          "name": "check_if_done",
          "taskReferenceName": "done_check",
          "type": "SWITCH",
          "evaluatorType": "value-param",
          "expression": "route",
          "inputParameters": {
            "route": "${plan.output.result.route}"
          },
          "decisionCases": {
            "needs_approval": [
              {
                "name": "human_approval",
                "taskReferenceName": "approval",
                "type": "HUMAN",
                "inputParameters": {
                  "plannedAction": "${plan.output.result.action}",
                  "arguments": "${plan.output.result.arguments}",
                  "reasoning": "${plan.output.result.reasoning}",
                  "goal": "${workflow.input.goal}"
                }
              },
              {
                "name": "execute_approved_tool",
                "taskReferenceName": "approved_tool_call",
                "type": "CALL_MCP_TOOL",
                "inputParameters": {
                  "mcpServer": "${workflow.input.mcpServerUrl}",
                  "method": "${plan.output.result.action}",
                  "arguments": "${plan.output.result.arguments}"
                }
              },
              {
                "name": "update_memory_approved",
                "taskReferenceName": "mem_update_approved",
                "type": "SET_VARIABLE",
                "inputParameters": {
                  "last_action": "${plan.output.result.action}",
                  "last_result": "${approved_tool_call.output.content}"
                }
              }
            ],
            "execute": [
              {
                "name": "execute_tool",
                "taskReferenceName": "tool_call",
                "type": "CALL_MCP_TOOL",
                "inputParameters": {
                  "mcpServer": "${workflow.input.mcpServerUrl}",
                  "method": "${plan.output.result.action}",
                  "arguments": "${plan.output.result.arguments}"
                }
              },
              {
                "name": "update_memory",
                "taskReferenceName": "mem_update",
                "type": "SET_VARIABLE",
                "inputParameters": {
                  "last_action": "${plan.output.result.action}",
                  "last_result": "${tool_call.output.content}"
                }
              }
            ],
            "done": [
              {
                "name": "save_answer",
                "taskReferenceName": "save_answer",
                "type": "SET_VARIABLE",
                "inputParameters": {
                  "final_answer": "${plan.output.result.answer}"
                }
              }
            ]
          },
          "defaultCase": []
        }
      ]
    }
  ],
  "outputParameters": {
    "answer": "${workflow.variables.final_answer}",
    "iterations": "${loop.output.iteration}",
    "last_action": "${workflow.variables.last_action}",
    "last_result": "${workflow.variables.last_result}"
  },
  "failureWorkflow": "agent_compensation_workflow"
}
```


## 什么让它达到生产就绪

### 每个步骤都是持久化检查点

在原生任务路径中，`DO_WHILE` 的每次迭代在开始下一次之前都被持久化。如果 agent 在第 15 次迭代（共 20 次）时崩溃，它会从第 15 次迭代恢复 — 而不是从头开始。每个 LLM prompt、响应、工具调用和人工决策都被记录。已部署的 Conductor Agents 提供同样的内部 Conductor 可见性，因为它们的图被编译成 Conductor 工作流。

对于 A2A 路径，持久化检查点是 `AGENT` 交接：Conductor 记录它的状态、重试和取消生命周期，以及返回的工件。远程 agent 的私有内部步骤仍由该远程服务拥有和观测。

### 人工审批是持久化关卡

`HUMAN` 任务无限期地暂停工作流。这次暂停熬过服务器重启、部署和基础设施变更。当审阅者通过 API 或 UI 批准时，工作流以审批负载作为任务输出恢复。没有轮询、没有超时（除非你配置了）、没有丢失的审批。

### 重试是自动且可配置的

每个工具调用（`CALL_MCP_TOOL`、`HTTP`、`SIMPLE`）都从它的[任务定义](../../documentation/configuration/taskdef.md)继承重试行为：

```json
{
  "name": "execute_tool",
  "retryCount": 3,
  "retryLogic": "EXPONENTIAL_BACKOFF",
  "retryDelaySeconds": 2,
  "responseTimeoutSeconds": 30
}
```

如果 MCP 服务器宕机，Conductor 按指数退避重试。LLM **不会**被再次调用 — 只有失败的工具调用会重试。

### 记忆跨迭代持久化

`SET_VARIABLE` 把累积的上下文存入工作流变量。这些变量被持久化到持久化存储中，并对后续每个任务可用。LLM 在每次迭代中接收动作和结果的完整历史。

### 预算上限防止 agent 失控

`loopCondition` 同时检查 agent 的 `done` 标志和迭代上限。你还可以在条件中检查 token 用量或成本。预算耗尽时，agent 干净地终止。

### 补偿处理副作用

如果 agent 在采取现实世界动作（发了一封邮件、创建了一条记录、扣了一笔款）之后失败，`failureWorkflow` 会自动运行补偿任务。补偿工作流收到完整的执行上下文：哪些动作成功了、哪些失败了，以及为什么。

### 可观测性是自动的

对于原生任务和编译后的 Conductor Agent 图，打开 Conductor UI 可以看到：

- 这次执行的确切任务图
- 每个 LLM prompt 和响应（点击任意 `LLM_CHAT_COMPLETE` 任务）
- 每个工具调用的输入、输出和耗时
- 每个人工审批的审批人和时间
- 迭代次数和循环状态
- 任何失败任务的重试历史
- 完整的工作流输入、输出和变量

对于远程 A2A agent，父工作流暴露持久化的 `AGENT` 任务 — 交接状态、重试和取消生命周期，以及返回的文本或工件。远程 agent 的内部图对其操作者保持私有，这正是让边界保持干净的原因。


## 扩展该模式

### 添加并行研究

用 `FORK_JOIN_DYNAMIC` 替换单个工具调用，以并行扇出到多个工具。在这个任务之前校验并限制 LLM 生成的输入；无界的计划不是安全的生产级扇出。

```json
{
  "name": "parallel_research",
  "taskReferenceName": "research",
  "type": "FORK_JOIN_DYNAMIC",
  "inputParameters": {
    "dynamicTasks": "${plan.output.result.parallel_tasks}",
    "dynamicTasksInput": "${plan.output.result.task_inputs}"
  },
  "dynamicForkTasksParam": "dynamicTasks",
  "dynamicForkTasksInputParamName": "dynamicTasksInput"
}
```

LLM 决定并行调用多少个工具、带什么输入。Conductor 在运行时创建分支。

### 添加反思 / 评估步骤

在工具执行之后插入一个 LLM-as-judge 来评估输出质量：

```json
{
  "name": "evaluate_result",
  "taskReferenceName": "evaluator",
  "type": "LLM_CHAT_COMPLETE",
  "inputParameters": {
    "llmProvider": "anthropic",
    "model": "claude-sonnet-4-20250514",
    "messages": [
      {
        "role": "system",
        "message": "Evaluate this result against the goal. Is it sufficient? Respond with JSON: {\"quality\": \"good\" or \"insufficient\", \"feedback\": \"...\"}"
      },
      {
        "role": "user",
        "message": "Goal: ${workflow.input.goal}\nResult: ${tool_call.output.content}"
      }
    ]
  }
}
```

如果评估者返回 `insufficient`，循环继续，并把反馈作为下一步规划的上下文。

### 添加长等待

插入一个 `WAIT` 任务进行基于时间的暂停（限流、冷却期、定时动作）：

```json
{
  "name": "wait_before_retry",
  "taskReferenceName": "cooldown",
  "type": "WAIT",
  "inputParameters": {
    "duration": "1 hour"
  }
}
```

等待是持久的。等待期间工作流不消耗资源。1 小时后 — 即使服务器在此期间重启过 — 工作流会恢复。

### 委派给专家 agent

当专家是一个 agent 运行时，使用 `AGENT`。已部署的 Conductor Agent 按名称调用：

```json
{
  "name": "delegate_to_planner",
  "taskReferenceName": "planner_agent",
  "type": "AGENT",
  "inputParameters": {
    "agentType": "conductor",
    "name": "specialist_planner",
    "prompt": "${workflow.input.goal}"
  }
}
```

当专家是独立部署的 A2A 服务时，使用 `agentType: "a2a"`：

```json
{
  "name": "delegate_to_researcher",
  "taskReferenceName": "research_agent",
  "type": "AGENT",
  "inputParameters": {
    "agentType": "a2a",
    "agentUrl": "${workflow.input.researchAgentUrl}",
    "text": "${plan.output.result.research_topic}"
  }
}
```

当专家是子工作流而非 agent 运行时，使用 `SUB_WORKFLOW`：

```json
{
  "name": "delegate_to_researcher",
  "taskReferenceName": "research_agent",
  "type": "SUB_WORKFLOW",
  "inputParameters": {
    "name": "research_agent_workflow",
    "version": 1,
    "input": {
      "topic": "${plan.output.result.research_topic}",
      "mcpServerUrl": "${workflow.input.mcpServerUrl}"
    }
  }
}
```

父工作流等待子工作流完成。如果它失败，父工作流的失败处理生效。其工作流树在 UI 中可观测。`START_WORKFLOW` 是相应的发后即忘选项；这两个任务都不是调用已部署或远程 agent 运行时的替代品。


## 原语对照

| "我需要我的 agent……" | 使用 | 为什么 |
|---|---|---|
| 等待工具回调 | `HUMAN` 任务或异步完成 | 持久化暂停。在 API 信号时带负载恢复。 |
| 休眠到重试窗口 | `WAIT` 任务 | 基于定时器的持久化暂停。零资源消耗。 |
| 在运行时选择下一个工具 | `DYNAMIC` 任务 | LLM 输出决定任务类型。在执行时解析。 |
| 并行调用多个工具 | `FORK/JOIN` 或 `FORK_JOIN_DYNAMIC` | 静态或运行时确定的并行。JOIN 等待所有分支。 |
| 循环直到目标达成 | `DO_WHILE` | 带检查点的循环。每次迭代都持久化。 |
| 调用已部署的专家 agent | `agentType: "conductor"` 的 `AGENT` | 运行一个命名的 Conductor Agent；其编译后的工作流图在 Conductor 中可检视。 |
| 交接给远程专家 agent | `agentType: "a2a"` 的 `AGENT` | 带父边界状态、生命周期和工件的持久化远程交接。 |
| 组合子工作流 | `SUB_WORKFLOW` 或 `START_WORKFLOW` | 等待式或发后即忘的子工作流组合；与调用 agent 运行时不同。 |
| 跨步骤累积上下文 | `SET_VARIABLE` | 工作流变量持久化到持久化存储。 |
| 评估输出质量 | 把 `LLM_CHAT_COMPLETE` 用作评估者 | 循环内的 LLM-as-judge 模式。 |
| 限制迭代或成本 | `DO_WHILE` `loopCondition` | 检查迭代次数、token 用量或成本。 |
| 失败时撤销副作用 | `failureWorkflow` | 工作流失败时补偿任务自动运行。 |
| 暂停等待人工审阅 | `HUMAN` 任务 | 无限期持久化暂停。熬过重启和部署。 |
| 在外部事件时恢复 | `HUMAN` 任务 + API/webhook | 外部系统带负载调用 Task Update API。 |
| 后处理结构化输出 | `INLINE`（JavaScript）或 `JSON_JQ_TRANSFORM` | 无需工作者的服务器端转换。 |


## 后续步骤

- **[Conductor Agents](conductor-agents.md)** — 围绕一个已部署的 SDK 编写的 agent 图使用此架构。
- **[框架 Agents](agent-framework-recipes.md)** — 受支持的框架路径和受维护的 SDK 示例。
- **[A2A 集成](a2a-integration.md)** — 交接给独立部署的 A2A agent，同时保留持久化的父工作流边界。
- **[AI Agent 的失败语义](failure-semantics.md)** — 精确的失败契约：崩溃、重试、重复和长等待下会发生什么。
- **[为什么 Agent 工作流选 Conductor](why-conductor.md)** — Conductor 为 agent 工作流开箱即用地提供什么。
- **[构建你的第一个 Agent 工作流图](first-ai-agent.md)** — 把 SDK 编写的 agent 与普通工作流任务组合起来。
- **[MCP 集成](mcp-guide.md)** — 连接任何 MCP 服务器，把工作流暴露为 MCP 工具。
- **[Token 效率](token-efficiency.md)** — 持久化执行如何节省 token、降低 LLM 成本。
